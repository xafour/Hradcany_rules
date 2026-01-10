# UC-1 Workflow - Upload & Analyze

**Tags:** `#gt-management` `#uc1` `#workflow`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active

---

## 🎯 CO JE UC-1

UC-1 (Upload & Analyze) je první Use Case v GT Management systému. Řeší základní scénář: uživatel má známku, chce ji přidat do systému. Nahraje fotografii nebo sken, systém známku automaticky analyzuje a navrhne klasifikaci. Ale protože automatická detekce není stoprocentní, klasifikace zatím není potvrzená - sken zůstává ve stavu "pending" a čeká na potvrzení v UC-2.

UC-1 je navržen tak, aby byl co nejjednodušší pro uživatele: nahraje jeden soubor (front scan), volitelně může přidat další soubory (back, certificate, block), systém udělá všechno ostatní. Analýza běží automaticky na pozadí - YOLO detekuje rámeček, warp normalizuje známku, embeddings se vypočítají, porovnání s referenčními skeny se provede. Uživatel dostane zpět návrh klasifikace typu "Pravděpodobně 500h TD I ZP 42, confidence HIGH".

---

## 🔄 WORKFLOW - 11 KROKŮ

UC-1 se skládá z 11 na sebe navazujících kroků. Každý krok má jasně definovaný vstup, výstup a zodpovědnost. Kroky jsou atomické - pokud kterýkoliv krok selže, celý workflow se zastaví a vrátí chybu.

### Krok 1: YOLO Frame Detection

Systém nejdřív musí najít rámeček známky na obrázku. Vstupní fotografie může obsahovat známku v různé pozici, natočenou, s okolím. YOLO model detekuje čtyři rohy známky (quad) s confidence score. Pokud je confidence pod 0.95, workflow se zastaví s chybou "Low confidence frame detection" - pravděpodobně známka není na obrázku dobře viditelná nebo je tam něco jiného než známka.

Tento krok je kritický protože všechny následující kroky závisí na správně detekovaném rámečku. Bez přesného quadu nemůžeme správně warpnout známku a embeddings by byly nekonzistentní s referenčními embeddingy.

**Výstup:** `quad = [(x1,y1), (x2,y2), (x3,y3), (x4,y4)]` + confidence score

---

### Krok 2: Warp to 1300×1100

Systém použije detekovaný quad k perspektivní transformaci známky na standardizovaný formát 1300×1100 pixelů. Tato velikost je určena tím, že referenční skeny v Ground Truth mají tuto velikost - musíme používat stejný formát pro konzistentní porovnání embeddingů.

Warp vytváříme POUZE pro zpracování (výpočet embeddingů), ne pro uložení. Do storage ukládáme VŽDY originální sken jak ho nahrál uživatel. Důvod: warp můžeme kdykoli zopakovat (máme uložený quad), ale originál po přepsání už nelze obnovit.

**Výstup:** `warped_bgr` - numpy array 1300×1100×3, BGR, uint8

---

### Krok 3: Compute SHA256

Systém vypočítá SHA256 hash ORIGINÁLU (ne warpu!). Tento hash slouží jako primární identifikátor skenu v systému - používá se pro deduplication, file naming, a path reconstruction.

Je kriticky důležité že hash počítáme z originálu, ne z warpu. Pokud bychom hashovali warp, dva uživatelé kteří nahrají stejný originální sken by dostali různé hashe (warp může být mírně jiný kvůli rounding errors v perspektivní transformaci). To by znemožnilo deduplication.

**Výstup:** `file_sha256` - string, 64 hex znaků

---

### Krok 4: Duplicate Detection

Systém zkontroluje jestli sken s tímto SHA256 už existuje v databázi (v tabulce `reference_front`). Pokud ano, nejde o nový sken, ale o duplicitní upload.

V případě duplicity systém NEPOKRAČUJE kroky 5-11, ale vrací existující scan_id a spouští re-analýzu LIVE (vypočítá embeddings z warpu aktuálního uploadu a porovná je s referenčními embeddingy, vrátí TOP-5 kandidátů). Důvod: možná původní analýza byla s horšími algoritmy, nebo uživatel chce vidět aktuální výsledky matchingu.

**Výstup:** `existing_scan_id` (pokud duplicate) nebo `None` (pokud nový)

---

### Krok 4b: Duplicate Re-analysis (volitelný)

Pokud je sken duplicate, systém spustí LIVE analýzu: vypočítá embeddings z aktuálního warpu, porovná je s referenčními embeddingy, vrátí TOP-5 nejpodobnějších kandidátů. Tato analýza NEukládá nic do databáze - je to jen pro informaci uživatele.

Pak se workflow ukončí s `status='duplicate'` a informací o tom, kdo sken původně uploadoval a jaké jsou aktuální match results.

---

### Krok 5: Save Files (SHA-based storage)

Systém uloží originální sken do SHA-based storage: `~/ProjektHradcany/<env>/gt_data/scans/<first2>/<sha256>_front.jpg`. První 2 znaky SHA určují subdirectory (00 až ff), což vytváří 256 subdirectories a umožňuje scalability na miliony souborů.

Pokud uživatel nahrál i další soubory (back, certificate, block), ty se uloží stejným způsobem: `<sha256>_back.jpg`, `<sha256>_cert.pdf`, `<sha256>_block.jpg`.

KRITICKÉ: Ukládáme ORIGINÁL, ne warp! Viz `tasks/bug_uc1_file_storage.md` pro známý bug v aktuální implementaci.

**Výstup:** Cesty k uloženým souborům

---

### Krok 6: DB Insert - reference_front

Systém vytvoří záznam v tabulce `reference_front`:
- `file_path` = NULL (rekonstrukce ze SHA)
- `file_sha256` = hash originálu
- `stamp_type_id` = NULL (pending, bude vyplněno v UC-2)
- `plate_id` = NULL
- `zp_no` = NULL
- `confirmed` = 0 (pending)
- `resolution_w`, `resolution_h` = rozměry ORIGINÁLU (ne warpu!)
- `uploaded_by` = username@hostname
- `suspect_flag` = 0
- `notes` = NULL nebo auto-detected info

Nullable metadata (stamp_type_id, plate_id, zp_no) representují stav "čeká na potvrzení". Viz `decisions/schema_v3_1_0_nullable_columns.md` pro odůvodnění.

**Výstup:** `scan_id` - INT, nově vytvořené ID

---

### Krok 7: DB Insert - user_uploads

Systém zaznamená vlastnictví skenu v tabulce `user_uploads`:
- `scan_id` = nově vytvořené ID
- `uploaded_by` = username@hostname
- `uploaded_at` = current timestamp
- `upload_source` = 'web_ui' nebo 'api' nebo 'batch_import'

Tato tabulka umožňuje multi-user systém kde každý uživatel vidí pouze svoje uploady (privacy-first). Pokud dva uživatelé nahrají stejný sken (stejné SHA256), v `user_uploads` budou dva záznamy ale v `reference_front` a storage jen jeden.

**Výstup:** user_upload_id

---

### Krok 8: DB Insert - inference_frames

Systém cachuje YOLO quad do tabulky `inference_frames`:
- `scan_id` = nově vytvořené ID
- `image_path` = cesta k originálu (pro referenci)
- `x1, y1, x2, y2, x3, y3, x4, y4` = quad coordinates
- `bx, by, bw, bh` = bounding box
- `img_w`, `img_h` = rozměry originálu
- `conf` = YOLO confidence score
- `model_id` = ID použitého YOLO modelu

Cachování quadu umožňuje skip YOLO detekce při budoucí re-analýze - YOLO je pomalé (~500ms), načtení quadu z cache je rychlé (~1ms). Toto je optimalizace pro performance.

**Výstup:** inference_frame_id

---

### Krok 9: DB Insert - scan_supplementary_files

Pokud uživatel nahrál additional files (back, cert, block), systém zaznamená jejich metadata do tabulky `scan_supplementary_files`:
- `scan_id` = nově vytvořené ID
- `file_type` = 'back' nebo 'cert' nebo 'block'
- `file_sha256` = hash daného souboru
- `uploaded_at` = timestamp

Tato tabulka umožňuje lazy loading - additional files se načítají jen když expert nebo systém potřebuje (například certifikát pro auto-approval check). UNIQUE constraint na (scan_id, file_type) zajišťuje max 1 back, 1 cert, 1 block per scan.

**Výstup:** supplementary_file_ids (pokud byly nahrány)

---

### Krok 10: Auto-detect Denomination (LIVE)

Systém spustí automatickou detekci denomination pomocí multi-phase pipeline: Embedding → OCR → HSV. Používá warped_bgr z kroku 2, RealEncoder pro výpočet embeddingů, a porovná je s referenčními embeddingy ze všech 26 denominations.

Tato detekce je LIVE - NEukládá se do databáze jako potvrzená klasifikace. Je to pouze návrh pro uživatele. Metadata zůstávají NULL v `reference_front` dokud uživatel nebo expert nepotvrdí klasifikaci v UC-2.

Auto-detect vrací:
- `denomination` - např. "500h"
- `confidence_level` - HIGH/MEDIUM/LOW/MANUAL_REVIEW
- `stamp_type_id` - ID z tabulky stamp_types
- `drawing_id` - ID kresby
- `similarity` - cosine similarity score
- `ocr_digits` - co OCR přečetlo
- `hsv_match` - jestli barva odpovídá

**Důvod LIVE detekce:** Kdyby se uložila do DB, vznikla by nekonzistence - sken by měl `stamp_type_id=5` ale stále `confirmed=0`. NULL metadata jasně říkají "ještě nepotvrzeno".

**Výstup:** Detection result dictionary

---

### Krok 11: Audit Log

Systém zaznamená celou operaci do audit logu (tabulka `gt_audit_log`):
- `scan_id` = nově vytvořené ID
- `action` = 'added_to_pending' nebo 'duplicate_detected'
- `performed_by` = username@hostname
- `timestamp` = current time
- `details` = JSON s info o auto-detection, confidence, atd.

Audit log slouží pro tracking všech změn v GT Management - kdo co kdy udělal. Umožňuje debugging a compliance (můžeme zpětně dohledat kdo nahrál problematický sken).

**Výstup:** audit_log_id

---

## 🎯 VÝSTUP CELÉHO UC-1

UC-1 vrací strukturovaný výsledek:

```python
{
    'status': 'new' | 'duplicate',
    'scan_id': int,
    'sha256': str,
    'auto_detected': {
        'denomination': str,
        'confidence_level': str,  # HIGH/MEDIUM/LOW
        'stamp_type_id': int,
        'similarity': float,
        ...
    },
    'top5_matches': [
        {
            'rank': int,
            'denomination': str,
            'plate_no': int,
            'zp_no': int,
            'similarity': float,
            'method': str  # 'embedding+ocr' nebo 'embedding_only'
        },
        ...
    ],
    'auto_approved': bool,  # True pokud splňuje podmínky
    'files_saved': {
        'front': Path,
        'back': Path | None,
        'cert': Path | None,
        'block': Path | None
    },
    'duplicate_info': dict | None  # Pokud duplicate
}
```

---

## 🔑 KLÍČOVÁ ROZHODNUTÍ

### LIVE detection vs DB storage

Rozhodli jsme se že auto-detection v UC-1 je LIVE (běží při každém uploadu) a NEukládá se jako potvrzená klasifikace. Metadata v `reference_front` zůstávají NULL dokud uživatel nebo expert nepotvrdí v UC-2.

**Důvod:** Automatická detekce není stoprocentní. Pokud bychom okamžitě vyplnili metadata, riskovali bychom chybné skeny v pending queue. NULL jasně říká "ještě nepotvrzeno" a umožňuje uživateli opravit chybu před potvrzením.

### Originál vs Warp storage

Rozhodli jsme se ukládat ORIGINÁL, ne warp. Warp vytváříme pouze dočasně pro zpracování (embeddings) a pak ho zahazujeme.

**Důvod:** Warp můžeme kdykoli znovu vytvořit (máme quad v cache), ale originál po přepsání už nelze obnovit. Ukládání originálu nám dává flexibilitu - v budoucnu můžeme použít jiný warp size nebo lepší algoritmus a re-analyzovat všechny skeny z originálů.

**ZNÁMÝ BUG:** Současná implementace (gt_upload_utils.py v1.6.0) ukládá warp místo originálu. Viz `tasks/bug_uc1_file_storage.md`.

### SHA256 z originálu

Rozhodli jsme se počítat SHA256 hash z ORIGINÁLU, ne z warpu.

**Důvod:** Deduplication. Dva uživatelé kteří nahrají stejný originální sken musí dostat stejný hash. Pokud bychom hashovali warp, mohli by dostat mírně odlišné hashe kvůli rounding errors při perspektivní transformaci.

### 11 kroků atomicky

Rozhodli jsme se rozdělit UC-1 na 11 jasně definovaných kroků. Každý krok má vstup, výstup, a zodpovědnost. Pokud kterýkoliv krok selže, celý workflow se zastaví.

**Důvod:** Debugovatelnost a testovatelnost. Když UC-1 selže, víme přesně ve kterém kroku. Můžeme testovat každý krok nezávisle. Můžeme přidat transaction rollback pokud krok 8 selže - stornujeme kroky 5-7.

---

## 🔗 SOUVISLOSTI

**Celkový koncept:**
- [decisions/gt_management_use_cases.md](./gt_management_use_cases.md) - Všechny Use Cases

**Následující krok:**
- UC-2: Confirm Classification - Uživatel/expert potvrdí klasifikaci

**Technická implementace:**
- [base/gt_workflows.md](../base/gt_workflows.md) - Workflow diagramy
- [base/pipeline_recognition.md](../base/pipeline_recognition.md) - Zlatý standard file handling
- [base/file_storage_architecture.md](../base/file_storage_architecture.md) - SHA256 storage

**Implementační status:**
- [tasks/uc1_implementation.md](../tasks/uc1_implementation.md) - Status, test results
- [tasks/bug_uc1_file_storage.md](../tasks/bug_uc1_file_storage.md) - Známý bug (warp vs originál)

**Další rozhodnutí:**
- [decisions/schema_v3_1_0_nullable_columns.md](./schema_v3_1_0_nullable_columns.md) - Proč NULL metadata
- [decisions/sha256_storage_strategy.md](./sha256_storage_strategy.md) - Proč SHA256 storage

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Status:** ✅ ACTIVE

---
