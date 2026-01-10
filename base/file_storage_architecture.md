# File Storage Architecture

**Tags:** `#file-storage` `#sha256` `#architecture`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active

---

## 🎯 ÚČEL

Tento dokument popisuje architekturu file storage systému v projektu Hradčany. Storage systém řeší jak a kde ukládáme skenované obrázky známek tak, aby byl systém scalable (umí zvládnout miliony souborů), multi-user safe (více uživatelů může nahrát stejný sken), a flexibilní (můžeme změnit storage lokaci bez migrace databáze).

Klíčovým konceptem je SHA256-based storage: každý soubor je pojmenovaný podle SHA256 hashu svého obsahu a uložený do subdirectory určeného prvními dvěma znaky hashu. Cesta k souboru se neukládá do databáze - místo toho se rekonstruuje z hashu když je potřeba. Tento přístup nám dává automatickou deduplication (dva stejné soubory = jeden storage záznam) a flexibilitu (můžeme přesunout storage bez update databáze).

---

## 📁 STRUKTURA STORAGE

Storage je organizovaný do této adresářové struktury:

```
~/ProjektHradcany/<env>/gt_data/scans/
├── 00/                              # Subdirectory pro hashe začínající "00"
│   ├── 00a1b2c3d4..._front.jpg      # Přední strana známky
│   ├── 00a1b2c3d4..._back.jpg       # Zadní strana (volitelné)
│   └── 00a1b2c3d4..._cert.pdf       # Certifikát (volitelné)
├── 01/                              # Subdirectory pro "01"
│   └── 01234567ab..._front.jpg
├── 02/
...
├── fe/
└── ff/                              # Subdirectory pro "ff"
    └── ffabcdef12..._front.jpg
```

Každý sken je uložený jako `<sha256>_<type>.<ext>`, kde:
- `<sha256>` je 64-znakový hexadecimální SHA256 hash obsahu souboru
- `<type>` je jeden z: `front`, `back`, `cert`, `block`
- `<ext>` je file extension: `.jpg`, `.png`, `.pdf`

První dva znaky hashu (`00` až `ff`) určují do kterého subdirectory soubor patří. To vytváří 256 subdirectories a rovnoměrně rozděluje soubory mezi ně.

---

## 🔑 PATH RECONSTRUCTION

Cesta k souboru se NEukládá do databáze (sloupec `file_path` je NULL). Místo toho se rekonstruuje z SHA256 hashu když potřebujeme přístup k souboru:

```python
def get_scan_path(sha256: str, file_type: str, env: str) -> Path:
    """
    Rekonstruuje cestu k souboru z SHA256 hashu.
    
    Args:
        sha256: SHA256 hash souboru (64 hex znaků)
        file_type: 'front', 'back', 'cert', nebo 'block'
        env: 'dev', 'prod', nebo 'sandbox'
    
    Returns:
        Absolutní cesta k souboru
    """
    root = Path.home() / 'ProjektHradcany' / env / 'gt_data' / 'scans'
    subdir = sha256[:2]  # První 2 znaky hashu
    ext = '.pdf' if file_type == 'cert' else '.jpg'
    filename = f"{sha256}_{file_type}{ext}"
    return root / subdir / filename
```

**Příklad:**
```python
sha256 = "abc123def456..."
path = get_scan_path(sha256, 'front', 'dev')
# Výsledek: /home/user/ProjektHradcany/dev/gt_data/scans/ab/abc123def456..._front.jpg
```

**Proč path reconstruction:**
Toto nám dává flexibilitu měnit storage strategii bez migrace databáze. Pokud přesuneme soubory na jiný disk nebo do cloudu, stačí změnit funkci `get_scan_path()` - databáze zůstane stejná.

---

## 📊 STORAGE TYPES

Storage rozlišuje čtyři typy souborů pro každý sken:

### Front (PRIMARY, required)

Přední strana známky - hlavní sken který obsahuje všechny vizuální rysy potřebné pro identifikaci. Tento soubor je VŽDY uložený jako ORIGINÁL jak ho nahrál uživatel - ne jako warp nebo jiná transformace.

**Proč originál:**
Warp můžeme kdykoli znovu vytvořit (máme uložený quad v `inference_frames`), ale originál po přepsání už nelze obnovit. Ukládání originálu nám dává možnost budoucí re-analýzy s lepšími algoritmy.

**File naming:** `<sha256>_front.jpg` nebo `_front.png`

### Back (OPTIONAL)

Zadní strana známky - používá se pro kontrolu vodotisku, typu papíru, nebo razítek. Není potřeba pro základní identifikaci, ale je užitečná pro expert review nebo pro rozlišení variant.

**File naming:** `<sha256>_back.jpg`

### Cert (OPTIONAL)

Certifikát autenticity - PDF nebo obrázek certifikátu od uznávaného experta. Pokud sken má certifikát, je kandidátem na auto-approval (automatické schválení do Ground Truth bez manuální expert review).

**File naming:** `<sha256>_cert.pdf` nebo `_cert.jpg`

### Block (OPTIONAL)

Bloková čtveřice - sken čtyř známek vedle sebe z jedné tiskové desky. Používá se pro plate reconstruction (určení které pozice byly vytištěné vedle sebe) a pro context poznání (kontrola že známka je skutečně z dané desky).

**File naming:** `<sha256>_block.jpg`

---

## 🔴 KRITICKÉ PRAVIDLO

### VŽDY ukládej ORIGINÁL, NE WARP!

```
✅ SPRÁVNĚ:
   User nahraje original.jpg (2400×3200)
   → Compute SHA256(original.jpg) → "abc123..."
   → Storage: ab/abc123..._front.jpg [ORIGINAL 2400×3200]
   → DB: file_sha256='abc123...', resolution_w=2400, resolution_h=3200

❌ ŠPATNĚ (BUG v UC-1):
   User nahraje original.jpg (2400×3200)
   → Warp to 1300×1100 → warped.jpg
   → Compute SHA256(warped.jpg) → "xyz789..."
   → Storage: xy/xyz789..._front.jpg [WARP 1300×1100]
   → DB: file_sha256='xyz789...', resolution_w=1300, resolution_h=1100
```

**Důsledky špatného workflow:**
- ❌ **Ztráta originálu** - nelze získat zpět, není možné re-analyzovat
- ❌ **Špatná deduplication** - dva uživatelé kteří nahrají stejný originál dostanou různé hashe (warp má rounding errors)
- ❌ **Nesprávná metadata** - resolution v databázi neodpovídá skutečnému skenu

**Známý bug:** Současná implementace UC-1 v `gt_upload_utils.py` v1.6.0 ukládá warp. Viz `tasks/bug_uc1_file_storage.md`.

---

## ✅ VÝHODY ARCHITEKTURY

### 1. Scalability

256 subdirectories umožňuje ukládat miliony souborů bez performance degradace. Při 1 milionu souborů bude v každém subdirectory průměrně ~4000 souborů, což je optimální pro filesystem performance.

### 2. Automatická deduplication

SHA256 hash automaticky detekuje duplicitní soubory. Když dva uživatelé nahrají STEJNÝ sken (byte-by-byte identický), dostanou STEJNÝ hash. Systém pozná duplicitu a neuloží soubor dvakrát.

V databázi budou dva záznamy v `user_uploads` (každý uživatel "vlastní" svůj upload), ale v `reference_front` a v storage jen jeden. To šetří místo a zajišťuje konzistenci.

### 3. Multi-user safe

Každý uživatel vidí "svůj" upload ve svém seznamu (díky `user_uploads`), ale fyzicky je uložen jen jeden soubor. Když uživatel A smaže svůj view, soubor zůstane pokud ho ještě vlastní uživatel B. Soubor se smaže ze storage až když ho smažou všichni vlastníci.

### 4. Flexibilita storage

Protože cesty rekonstruujeme z hashe, můžeme změnit storage lokaci bez migrace databáze. Příklady:

**Přesun na jiný disk:**
```python
# Změna root:
root = Path('/mnt/fast_ssd') / 'hradcany_storage'
```

**Budoucí cloud storage:**
```python
def get_scan_url(sha256: str, file_type: str) -> str:
    return f"https://storage.hradcany.cz/{sha256[:2]}/{sha256}_{file_type}.jpg"
```

Databáze zůstává stejná, jen změníme jak konstruujeme cesty/URLs.

---

## 🔗 DB INTEGRATION

Storage je integrovaný s databází přes tyto tabulky:

### reference_front

```sql
CREATE TABLE reference_front (
    id INTEGER PRIMARY KEY,
    file_path TEXT,              -- NULL (rekonstrukce z SHA)
    file_sha256 TEXT UNIQUE,     -- PRIMARY identifier
    resolution_w INTEGER,        -- Rozměry ORIGINÁLU
    resolution_h INTEGER,
    ...
);
```

### scan_supplementary_files

```sql
CREATE TABLE scan_supplementary_files (
    id INTEGER PRIMARY KEY,
    scan_id INTEGER,             -- FK → reference_front.id
    file_type TEXT,              -- 'back', 'cert', 'block'
    file_sha256 TEXT,            -- Hash supplementary souboru
    ...
);
```

Supplementary files mají VLASTNÍ SHA256 hash (ne stejný jako front) protože jsou to samostatné soubory. Například certifikát může být PDF s jiným obsahem než sken známky.

---

## 🔗 SOUVISLOSTI

**Zlatý standard:**
- [base/pipeline_recognition.md](./pipeline_recognition.md) - Správný file handling workflow

**Rozhodnutí:**
- [decisions/sha256_storage_strategy.md](../decisions/sha256_storage_strategy.md) - Proč SHA256 storage
- [decisions/uc1_workflow.md](../decisions/uc1_workflow.md) - Jak UC-1 ukládá soubory

**Známý bug:**
- [tasks/bug_uc1_file_storage.md](../tasks/bug_uc1_file_storage.md) - KRITICKÝ bug (warp vs originál)

---

**GOLDEN RULE: Storage = ORIGINÁL, Warp = TEMP processing only!**

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Status:** ✅ ACTIVE

---
