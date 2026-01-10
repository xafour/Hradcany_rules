# Schema v3.1.0 - Nullable Columns

**Tags:** `#database` `#schema` `#v3.1.0` `#migration`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active

---

## 🎯 ROZHODNUTÍ

Rozhodli jsme se změnit databázové schema tak, aby sloupce `stamp_type_id`, `plate_id`, `zp_no` a `file_path` v tabulce `reference_front` byly nullable (mohly obsahovat NULL hodnotu místo toho, aby vyžadovaly vždy nějakou hodnotu). Tato změna byla učiněna v rámci migrace na schema verzi 3.1.0 dne 2025-12-31.

Toto rozhodnutí přímo souvisí s implementací GT Management systému, konkrétně s dvoufázovým workflow Upload → Confirm. V UC-1 uživatel nahraje známku a systém ji automaticky analyzuje, ale tato analýza zatím není potvrzená. Teprve v UC-2 uživatel nebo expert potvrdí správnost klasifikace. Během pending stavu (mezi UC-1 a UC-2) potřebujeme v databázi reprezentovat stav "metadata jsou zatím neznámá" nebo "čekají na potvrzení". NULL hodnota je pro tento účel přirozené řešení.

---

## 🔍 KONTEXT A DŮVODY

### Problém s NOT NULL sloupci

Před verzí 3.1.0 byly sloupce `stamp_type_id`, `plate_id`, `zp_no` definované jako NOT NULL - musely vždy obsahovat nějakou hodnotu. To fungovalo dobře pro baseline skeny, kde jsme metadata znali dopředu (načítali jsme je z katalogu knihtisk.org a hned při vložení do databáze jsme věděli jaká známka to je).

Ale když jsme začali implementovat GT Management, narazili jsme na problém: uživatel nahraje známku přes UC-1, systém ji automaticky analyzuje a navrhne klasifikaci (například "500h TD I ZP 42"), ale tato klasifikace zatím není potvrzená. Co máme uložit do databáze? Máme použít navrženou klasifikaci? Ale co když je špatně? Máme použít placeholder hodnoty (například stamp_type_id=0)? Ale to by bylo hack řešení a vedlo by to k nekonzistenci - v databázi by byly záznamy kde stamp_type_id=0 znamená "placeholder" a záznamy kde to znamená skutečnou známku.

### NULL jako přirozená reprezentace "neznámého"

NULL hodnota v SQL přirozeně reprezentuje stav "neznámé" nebo "není k dispozici". Když nastavíme `stamp_type_id = NULL`, jasně říkáme "tento sken zatím nemá potvrzenou klasifikaci". Není to placeholder, není to chyba, je to explicitní stav "čeká na vyplnění".

Tento přístup má několik výhod: (1) jasná sémantika - NULL znamená pending, konkrétní hodnota znamená confirmed, (2) žádné placeholder hodnoty které by mohly kolidovat se skutečnými daty, (3) snadné SQL dotazy - `WHERE stamp_type_id IS NULL` vrátí všechny pending skeny, (4) konzistence s dalšími částmi systému kde NULL reprezentuje "optional" nebo "not yet set".

### file_path jako rekonstruovatelné

Sloupec `file_path` jsme také udělali nullable, ale z jiného důvodu. Rozhodli jsme se že cesta k souboru není primární identifikátor - tím je `file_sha256`. Cesta se může měnit (můžeme přesunout storage do jiného adresáře, jiného disku, do cloudu), ale SHA256 hash zůstává stejný.

Když je `file_path = NULL`, znamená to že cestu můžeme kdykoliv rekonstruovat z SHA256: `~/ProjektHradcany/<env>/gt_data/scans/<first2>/<sha256>_front.jpg`. Toto nám dává flexibilitu - můžeme změnit storage strategii bez nutnosti updatovat tisíce záznamů v databázi.

---

## 💡 DŮSLEDKY

### Workflow Upload → Confirm

Nullable columns umožňují čistou implementaci dvoufázového workflow:

**UC-1 (Upload):**
```sql
INSERT INTO reference_front (
    file_sha256,
    stamp_type_id,  -- NULL
    plate_id,       -- NULL
    zp_no,          -- NULL
    confirmed,      -- 0 (pending)
    resolution_w,
    resolution_h,
    uploaded_by
) VALUES (
    'abc123...',
    NULL,  -- Čeká na potvrzení
    NULL,
    NULL,
    0,
    2400,
    3200,
    'milan@zenbook'
);
```

**UC-2 (Confirm):**
```sql
UPDATE reference_front
SET stamp_type_id = 5,    -- 500h
    plate_id = 10,        -- TD I
    zp_no = 42,           -- Pozice 42
    confirmed = 1         -- Verified
WHERE id = 150;
```

Tento pattern je mnohem čistší než používání placeholder hodnot. Stav skenu je okamžitě jasný z metadat - pokud jsou NULL, sken je pending.

### SQL dotazy

Nullable columns umožňují jednoduché dotazy pro různé stavy:

```sql
-- Všechny pending skeny (čekají na confirm)
SELECT * FROM reference_front
WHERE confirmed = 0 AND stamp_type_id IS NULL;

-- Všechny verified skeny
SELECT * FROM reference_front  
WHERE confirmed = 1 AND stamp_type_id IS NOT NULL;

-- Problematické záznamy (confirmed=1 ale metadata chybí)
SELECT * FROM reference_front
WHERE confirmed = 1 AND stamp_type_id IS NULL;
-- Toto by nemělo vrátit žádný výsledek!
```

### Migrace z v3.0.0 na v3.1.0

Změna NOT NULL → nullable vyžadovala rebuild tabulky, protože SQLite nepodporuje `ALTER COLUMN DROP NOT NULL`. Museli jsme použít migrační strategii: (1) vytvořit novou tabulku s nullable columns, (2) zkopírovat všechna data, (3) smazat starou tabulku, (4) přejmenovat novou tabulku.

Všechny existující záznamy (6,800 baseline skenů) měly metadata vyplněná, takže migrace neměla dopad na data - pouze změnila schema aby umožnilo NULL hodnoty pro budoucí pending skeny.

---

## 🔧 TECHNICKÉ DETAILY

### Schema před změnou (v3.0.0):

```sql
CREATE TABLE reference_front (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_path TEXT NOT NULL,
    stamp_type_id INTEGER NOT NULL,
    plate_id INTEGER NOT NULL,
    zp_no INTEGER NOT NULL,
    confirmed INTEGER DEFAULT 1,
    ...
);
```

### Schema po změně (v3.1.0):

```sql
CREATE TABLE reference_front (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_path TEXT,              -- NULLABLE (reconstruction)
    stamp_type_id INTEGER,       -- NULLABLE (pending state)
    plate_id INTEGER,            -- NULLABLE
    zp_no INTEGER,               -- NULLABLE
    confirmed INTEGER DEFAULT 0, -- Changed: 1 → 0
    ...
);
```

Všimni si také změny `confirmed DEFAULT 0` (místo 1) - nově nahrané skeny jsou ve výchozím stavu pending, ne verified.

---

## 🔗 SOUVISLOSTI

**Celkový koncept:**
- [decisions/gt_management_use_cases.md](./gt_management_use_cases.md) - Proč dvoufázový workflow

**Technická dokumentace:**
- [base/database_schema.md](../base/database_schema.md) - Kompletní DB schema v3.1.0

**Další rozhodnutí:**
- [decisions/uc1_workflow.md](./uc1_workflow.md) - Jak UC-1 používá NULL metadata

**Implementace:**
- Migration script: `tools/migrate_schema_v3_1_0.sql`
- Verified: 2025-12-31, all 6,800 baseline scans preserved

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Status:** ✅ ACTIVE

---
