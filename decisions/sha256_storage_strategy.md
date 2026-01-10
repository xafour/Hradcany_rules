# SHA256 Storage Strategy

**Tags:** `#file-storage` `#sha256` `#deduplication`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active

---

## 🎯 ROZHODNUTÍ

Rozhodli jsme se používat SHA256 hash jako primární identifikátor souborů v GT Management systému. Soubory se ukládají do struktury `~/ProjektHradcany/<env>/gt_data/scans/<first2>/<sha256>_<type>.<ext>`, kde `<first2>` jsou první dva znaky SHA256 hashu (00 až ff), což vytváří 256 subdirectories. Cesta k souboru se NEukládá do databáze - místo toho se rekonstruuje z SHA256 hashu když je potřeba.

Toto rozhodnutí řeší několik problémů najednou: (1) automatická deduplication - dva uživatelé kteří nahrají stejný sken dostanou stejný hash a systém pozná duplicitu, (2) scalability - 256 subdirectories umožňuje ukládat miliony souborů bez performance degradace, (3) flexibilita - můžeme změnit storage lokaci bez update databáze, (4) multi-user safe - každý uživatel "vlastní" svůj upload, ale fyzicky je uložen jen jeden soubor.

SHA256 hash počítáme VŽDY z originálního souboru, ne z transformovaných verzí (warp). Důvod: dva uživatelé kteří nahrají stejný originální sken musí dostat stejný hash. Pokud bychom hashovali transformaci, mohli by dostat mírně odlišné hashe kvůli rounding errors a deduplication by nefungovala.

---

## 🔍 KONTEXT A DŮVODY

### Problém s klasickým storage

Klasický přístup k ukládání souborů je: každý soubor má unikátní jméno (například `500hal_TD1_zp042.jpg`) a ukládá se do jednoho adresáře nebo do adresářové struktury podle typu známky. Cesta se ukládá do databáze v `file_path` sloupci.

Tento přístup má několik problémů: (1) co když dva uživatelé nahrají stejný sken s různými jmény? Uloží se dvakrát a ztratíme místo. (2) co když změníme naming convention nebo přesuneme soubory do jiného adresáře? Musíme updatovat tisíce záznamů v databázi. (3) jak zajistit že file names jsou unique? Musíme generovat artificial IDs nebo checkovat kolize. (4) jak efektivně ukládat miliony souborů? Jeden directory s milionem souborů má špatnou performance.

### SHA256 jako identifikátor

SHA256 hash je 256-bitové (64 hex znaků) číslo které prakticky jednoznačně identifikuje obsah souboru. Pravděpodobnost kolize (dva různé soubory mají stejný hash) je astronomicky nízká - méně než že tě zasáhne asteroid. Pro praktické účely můžeme považovat SHA256 za unique identifikátor.

Když použijeme SHA256 jako identifikátor souboru, řešíme problém (1): dva uživatelé kteří nahrají STEJNÝ sken (byte-by-byte identický) dostanou STEJNÝ hash. Systém pozná že jde o duplicitu a neuloží soubor dvakrát. V databázi vytvoří dva záznamy v `user_uploads` (každý uživatel "vlastní" svůj upload), ale v `reference_front` a v storage je jen jeden záznam.

### 256 subdirectories strategie

Ukládat milion souborů do jednoho adresáře má špatnou performance - filesystémy nejsou optimalizované pro tolik souborů v jednom directory. Řešení: rozdělit soubory do subdirectories.

Používáme první 2 znaky SHA256 hashu (00 až ff) jako název subdirectory. To vytváří 256 subdirectories a soubory se rovnoměrně rozdělí mezi ně (SHA256 je cryptographically random). Při milionu souborů bude v každém subdirectory průměrně ~4000 souborů, což je výborná performance.

Další výhoda: strategie je deterministická. Když máme hash `abc123...`, víme že soubor je v `ab/abc123..._front.jpg`. Nemusíme hledat, nemusíme indexovat, prostě rekonstruujeme cestu.

### Path reconstruction vs storage

Rozhodli jsme se NEUKLÁDAT cestu k souboru do databáze (`file_path = NULL`). Místo toho ji rekonstruujeme z SHA256 hashu když potřebujeme:

```python
def get_scan_path(sha256: str, file_type: str, env: str) -> Path:
    root = Path.home() / 'ProjektHradcany' / env / 'gt_data' / 'scans'
    subdir = sha256[:2]  # První 2 znaky
    filename = f"{sha256}_{file_type}.jpg"
    return root / subdir / filename
```

To nám dává flexibilitu: můžeme změnit root path (přesunout storage na jiný disk, do cloudu) bez update databáze. Stačí změnit funkci `get_scan_path()` a všechno funguje. Také ušetříme místo v databázi - místo ~100 znaků cesty ukládáme jen 64 znaků hash.

---

## 💡 DŮSLEDKY

### Automatická deduplication

Když uživatel nahraje sken, systém vypočítá SHA256 hash a zkontroluje jestli už existuje v databázi:

```python
file_sha256 = compute_sha256(uploaded_file)
existing = db.execute(
    "SELECT id FROM reference_front WHERE file_sha256 = ?",
    (file_sha256,)
).fetchone()

if existing:
    # Duplicita - sken už existuje
    return {'status': 'duplicate', 'scan_id': existing['id']}
else:
    # Nový sken - ulož soubor a vytvoř záznam
    save_file(uploaded_file, file_sha256)
    scan_id = insert_scan(file_sha256, ...)
    return {'status': 'new', 'scan_id': scan_id}
```

Toto je elegantní řešení - nemusíme porovnávat soubory pixel-by-pixel, stačí porovnat hashe. A protože hash počítáme z originálu, garantujeme že dva stejné originály budou mít stejný hash.

### Multi-user ownership

Deduplication je multi-user safe. Když uživatel A nahraje sken `abc123...` a později uživatel B nahraje STEJNÝ sken:

**V databázi:**
```sql
-- reference_front (jeden záznam)
id=150, file_sha256='abc123...', confirmed=1, ...

-- user_uploads (dva záznamy)
id=1, scan_id=150, uploaded_by='userA'
id=2, scan_id=150, uploaded_by='userB'
```

**V storage:**
```
gt_data/scans/ab/abc123..._front.jpg  (jeden soubor)
```

Každý uživatel vidí "svůj" upload ve svém seznamu (díky záznamu v `user_uploads`), ale fyzicky je uložen jen jeden soubor. Když uživatel A smaže svůj view, záznam v `user_uploads` se smaže, ale soubor zůstane (protože ho ještě vlastní user B).

### Scalability

256 subdirectories umožňuje škálovat na miliony souborů. Při 1 milionu souborů:
- Průměrně ~4000 souborů per subdirectory
- Filesystem performance zůstává dobrá
- Lookup je O(1) - hash → subdirectory → soubor

Pokud bychom v budoucnu potřebovali ještě lepší scalability (desítky milionů souborů), můžeme přidat další úroveň: `<first2>/<next2>/<sha256>_front.jpg` (65,536 subdirectories).

### Flexibilita storage

Protože cesty rekonstruujeme z hashe, můžeme změnit storage strategii bez migrace databáze:

**Změna root path:**
```python
# Původně:
root = Path.home() / 'ProjektHradcany' / env / 'gt_data' / 'scans'

# Nový disk:
root = Path('/mnt/fast_ssd') / 'hradcany_storage'
```

**Cloud storage (budoucnost):**
```python
# S3 bucket:
def get_scan_url(sha256: str, file_type: str) -> str:
    return f"s3://hradcany-prod/{sha256[:2]}/{sha256}_{file_type}.jpg"
```

Databáze zůstává stejná, jen změníme jak konstruujeme cesty/URLs.

---

## 🔧 TECHNICKÉ DETAILY

### Výpočet SHA256

```python
import hashlib

def compute_sha256(file_path: Path) -> str:
    """
    Vypočítá SHA256 hash souboru.
    Čte po chunkcích pro memory efficiency.
    """
    sha = hashlib.sha256()
    with open(file_path, 'rb') as f:
        while chunk := f.read(8192):
            sha.update(chunk)
    return sha.hexdigest()  # 64 hex znaků
```

KRITICKÉ: Počítáme hash z ORIGINÁLU, ne z transformací (warp)! Viz `tasks/bug_uc1_file_storage.md` pro známý bug kde současná implementace hashuje warp.

### Storage struktura

```
~/ProjektHradcany/dev/gt_data/scans/
├── 00/
│   ├── 00a1b2c3..._front.jpg
│   ├── 00a1b2c3..._back.jpg
│   └── 00d4e5f6..._front.jpg
├── 01/
│   └── 01234567..._front.jpg
├── 02/
...
├── fe/
└── ff/
    └── ffabcdef..._front.jpg
```

Každý subdirectory (00 až ff) obsahuje soubory jejichž hash začíná těmito dvěma znaky. V každém souboru je suffix `_front`, `_back`, `_cert`, nebo `_block` podle typu.

### File types

- `_front.jpg` - Přední strana známky (REQUIRED, primary)
- `_back.jpg` - Zadní strana (OPTIONAL, pro vodotisk check)
- `_cert.pdf` - Certifikát autenticity (OPTIONAL, pro auto-approval)
- `_block.jpg` - Bloková čtveřice (OPTIONAL, pro plate reconstruction)

---

## 🔗 SOUVISLOSTI

**Používáno v:**
- UC-1 workflow - Deduplication check, file storage
- UC-4 batch import - Bulk file handling

**Technická dokumentace:**
- [base/file_storage_architecture.md](../base/file_storage_architecture.md) - Detaily storage systému

**Další rozhodnutí:**
- [decisions/uc1_workflow.md](./uc1_workflow.md) - Jak UC-1 používá SHA256 storage
- [decisions/schema_v3_1_0_nullable_columns.md](./schema_v3_1_0_nullable_columns.md) - Proč file_path je nullable

**Známý bug:**
- [tasks/bug_uc1_file_storage.md](../tasks/bug_uc1_file_storage.md) - SHA256 musí být z originálu, ne warpu!

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Status:** ✅ ACTIVE

---
