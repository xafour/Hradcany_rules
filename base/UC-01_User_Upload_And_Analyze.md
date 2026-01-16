# UC-01: User Upload & Analyze

**Tags:** `#gt-management` `#user-workflow` `#upload` `#auto-detect`  
**Verze:** 1.0.0  
**Datum:** 2026-01-15  
**Status:** ACTIVE

---

## 🎯 ÚČEL

Tento Use Case popisuje proces, kterým běžný uživatel nahraje sken známky do systému a získá automatickou analýzu s návrhem klasifikace. Jedná se o vstupní bod do GT (Ground Truth) Management systému, kde uživatel ještě nemá potvrzenou identifikaci známky, ale systém mu poskytne TOP-5 nejpodobnějších kandidátů na základě strojového učení.

Use Case řeší následující potřeby: uživatel má naskenovanou známku (případně i její zadní stranu, certifikát nebo blokové čtveřice) a chce zjistit, o jakou známku se jedná. Systém mu poskytne okamžitou analýzu bez nutnosti manuálního procházení katalogů nebo expertní znalosti. Výsledek analýzy slouží jako základ pro následné potvrzení klasifikace (UC-02), kde uživatel buď souhlasí s návrhem systému, nebo jej opraví.

Klíčovým principem je privacy-first přístup - každý uživatel vidí pouze své vlastní nahrané skeny a systém nikdy neodhaluje informace o uploadu jiných uživatelů. Pokud dva různí uživatelé nahrají stejný sken (identifikovaný pomocí SHA256 hash), každý z nich ho vidí jako "nový ve svém view" bez informace o tom, že již existuje v databázi.

---

## 📋 PŘEHLED

Use Case UC-01 se skládá z několika fází, které probíhají automaticky po uploadu souboru uživatelem. Systém nejprve detekuje rámeček známky pomocí YOLO modelu, vypočítá SHA256 hash z originálního nahraného obrázku a uloží originální soubory do SHA-based adresářové struktury.

Následně systém vytvoří záznamy v databázových tabulkách `reference_front` (s `stamp_type_id=NULL` a `confirmed=0`, protože uživatel ještě nepotvrdil klasifikaci), `user_uploads` (pro tracking vlastnictví) a případně `scan_supplementary_files` (pokud byly nahrány dodatečné soubory jako zadní strana nebo certifikát). 

Poté se spustí auto-detection denominace, který provede dočasnou perspektivní korekci (warp) pro ML analýzu pomocí embedding analýzy, OCR verifikace a HSV color matchingu. Tento warp je POUZE pro analýzu a NEUKLÁDÁ se - v storage zůstává originální upload. Auto-detection vrátí TOP-5 nejpodobnějších kandidátů z referenční databáze.

Důležité je, že v tomto kroku se NEUKLÁDAJÍ embeddingy do tabulky `reference_embeddings`, protože systém ještě nezná `stamp_type_id` (a tudíž ani `drawing_id`), což je nutné pro určení, které crops (výřezy) má z obrázku extrahovat. Embeddingy budou vypočítány až v UC-02 po potvrzení klasifikace uživatelem.

---

## 🏗️ ARCHITEKTURA / STRUKTURA

Workflow UC-01 je implementováno jako posloupnost operací, které transformují nahraný obrázek na strukturovaná data v databázi a souborovém systému. Celý proces je navržen tak, aby byl idempotentní - pokud uživatel nahraje stejný sken vícekrát (nebo více uživatelů nahraje stejný sken), systém to detekuje podle SHA256 hash a nevytváří duplicitní soubory, pouze přidává záznam do `user_uploads` tabulky.

### Datový tok

Vstupem jsou soubory nahrané uživatelem: povinný `front_file` (líc známky) a volitelné `back_file` (rub), `cert_file` (certifikát autenticity) a `block_file` (bloková čtveřice pro kontext). Systém zpracuje front_file pomocí YOLO detekce a vypočítá SHA256 hash z originálního uploadu. Tento hash slouží jako primární identifikátor skenu a je použit pro pojmenování VŠECH souborů souvisejících s tímto skenem.

Výstupem jsou nové záznamy v databázi a uložené originální soubory ve formátu:
```
/gt_data/scans/{sha[:2]}/{sha}_front.jpg     # Originál front
/gt_data/scans/{sha[:2]}/{sha}_back.jpg      # Originál back
/gt_data/scans/{sha[:2]}/{sha}_cert.jpg      # Originál cert
/gt_data/scans/{sha[:2]}/{sha}_block.jpg     # Originál block
```

**KRITICKÉ:** Všechny soubory používají STEJNÝ SHA prefix (vypočítaný z front originálu), což znamená že všechny soubory pro jeden sken jsou pohromadě v jednom subdirectory. To umožňuje snadné načítání všech souvisejících souborů.

Všechny soubory jsou uloženy jako originály (ne transformované). Perspektivní korekce (warp) se provede pouze dočasně pro ML analýzu v kroku auto-detection, ale NEULOŽÍ se - originál musí být zachován pro budoucí re-analýzu.

### Databázové operace

Use Case provádí následující INSERT operace:

1. **reference_front** - hlavní záznam skenu s následujícími hodnotami:
   - `front_sha256` = vypočítaný hash (UNIQUE constraint)
   - `stamp_type_id` = NULL (čeká na UC-02)
   - `plate_id` = 0 (default, čeká na UC-02)
   - `zp_no` = 0 (default, čeká na UC-02)
   - `confirmed` = 0 (pending expert review)

2. **user_uploads** - tracking vlastnictví:
   - `scan_id` = ID z reference_front
   - `uploaded_by` = username@hostname
   - `upload_source` = 'web_ui' | 'api' | 'batch_import'

3. **inference_frames** - YOLO frame cache:
   - `scan_id` = ID z reference_front
   - `x1, y1, x2, y2, x3, y3, x4, y4` = souřadnice rohů detekovaného rámečku
   - `conf` = confidence YOLO detekce

4. **scan_supplementary_files** - dodatečné soubory (pokud byly nahrány):
   - Pro každý typ ('back', 'cert', 'block') samostatný záznam

---

## 🔄 WORKFLOW / JAK TO FUNGUJE

Workflow UC-01 probíhá v následujících krocích. Uživatel iniciuje proces nahráním souboru (nebo souborů) přes webové rozhraní, API endpoint nebo batch import.

### Krok 1: Přijetí souborů

Systém přijme soubory od uživatele a provede základní validaci formátu. Front_file musí být přítomen a musí být validní obrázkový formát (JPG, PNG). Ostatní soubory (back, cert, block) jsou volitelné.

**Funkce (bude implementována):**
```python
# Konceptuální - funkce bude implementována
process_user_upload(
    front_file,
    back_file=None,
    cert_file=None, 
    block_file=None,
    uploaded_by: str,
    upload_source: str = 'web_ui'
) -> dict
```

### Krok 2: YOLO detekce rámečku

Systém načte front_file a spustí YOLO model pro detekci rámečku známky. YOLO vrátí oriented bounding box (OBB) definovaný čtyřmi rohy: top-left (TL), top-right (TR), bottom-right (BR), bottom-left (BL). Tyto souřadnice jsou uloženy do `inference_frames` tabulky jako cache pro případné budoucí použití.

**Důležité:** YOLO detekce může selhat pokud je obrázek příliš rozmazaný, má špatné osvětlení nebo známka není viditelná. V takovém případě workflow končí chybou a uživatel je informován o nutnosti nahrát kvalitnější sken.

**Technické detaily:**
```python
# Používá existující yolo_utils.py
from common.yolo_utils import detect_frame_yolo

quad, conf, bbox = detect_frame_yolo(front_file, model_path, conf_threshold=0.5)
# quad: [(x1,y1), (x2,y2), (x3,y3), (x4,y4)] nebo None
# conf: Confidence score (0.0-1.0) nebo None
# bbox: (bx, by, bw, bh) nebo None
```

### Krok 3: Výpočet SHA256 hash z ORIGINÁLU

Systém vypočítá SHA256 hash z ORIGINÁLNÍHO front_file obrázku (před jakoukoliv transformací). Tento hash slouží jako jedinečný identifikátor skenu napříč celým systémem. 

**KRITICKÉ:** Hash se počítá z ORIGINÁLU, NE z warpované verze! Důvod: dva uživatelé kteří nahrají STEJNÝ originální sken musí dostat STEJNÝ hash. Pokud bychom hashovali warp, mohli by dostat mírně odlišné hashe kvůli rounding errors při transformaci a deduplication by nefungovala.

**Technické detaily:**
```python
import hashlib

def compute_sha256(file_path: str) -> str:
    sha = hashlib.sha256()
    with open(file_path, 'rb') as f:
        while chunk := f.read(8192):
            sha.update(chunk)
    return sha.hexdigest()  # 64 hex znaků

# SPRÁVNĚ: SHA z originálu
sha256 = compute_sha256(original_upload)  # PŘED warpem!
```

**Kontrola duplicity:** Systém zkontroluje, zda SHA256 již existuje v tabulce `reference_front`. Pokud ano, sken již byl nahrán (buď tímto uživatelem nebo jiným uživatelem). V takovém případě:
- Pokud tento uživatel již má tento sken ve svém view (existuje v `user_uploads`), workflow končí s hláškou "Již jste tento sken nahráli"
- Pokud tento sken nahrál jiný uživatel, systém přidá záznam do `user_uploads` pro aktuálního uživatele a vrátí mu "Nový sken ve vašem view" (PRIVACY: uživatel NIKDY nevidí, že sken nahrál někdo jiný)

### Krok 4: Uložení originálního front_file do filesystému

Systém uloží ORIGINÁLNÍ front_file (ne warpovanou verzi!) do SHA-based adresářové struktury. První dva znaky SHA slouží jako název subdirectory (vytváří 256 top-level adresářů pro distribuci zátěže).

**KRITICKÉ: Ukládá se ORIGINÁL, ne warp!**

Warp je pouze dočasná transformace používaná pro ML analýzu (embedding extraction). Originální sken musí být zachován protože:
- Warp můžeme kdykoli znovu vytvořit (máme quad v `inference_frames`)
- Originál po přepsání nelze obnovit
- Budoucí re-analýza s lepšími algoritmy vyžaduje originál

**Soubory jsou uloženy takto:**
- `{sha}_front.jpg` - ORIGINÁLNÍ upload (ne warp!)
- `{sha}_back.jpg` - Originální zadní strana (pokud byla nahrána)
- `{sha}_cert.jpg` - Originální certifikát (pokud byl nahrán)
- `{sha}_block.jpg` - Originální bloková čtveřice (pokud byla nahrána)

**DŮLEŽITÉ:** Všechny soubory používají STEJNÝ SHA prefix (vypočítaný z front originálu)! To znamená že všechny soubory pro jeden sken jsou pohromadě v JEDNOM subdirectory.

Back, cert a block soubory mají vlastní SHA hash uložený v `scan_supplementary_files.file_sha256` (pro integritu), ale pojmenování souboru používá SHA z front_file pro snadné načítání.

**Technické detaily:**
```python
# SHA z front originálu
sha256 = compute_sha256(original_front)  # např. "abc123..."

# Target directory
target_dir = f"/gt_data/scans/{sha256[:2]}"  # např. "ab/"
os.makedirs(target_dir, exist_ok=True)

# Uložit ORIGINÁL front (ne warp!)
shutil.copy(original_front, f"{target_dir}/{sha256}_front.jpg")

# Supplementary files - pojmenované podle FRONT SHA!
if back_file:
    back_sha = compute_sha256(back_file)  # Pro DB integritu
    shutil.copy(back_file, f"{target_dir}/{sha256}_back.jpg")  # Jméno z FRONT SHA!
    
if cert_file:
    cert_sha = compute_sha256(cert_file)
    shutil.copy(cert_file, f"{target_dir}/{sha256}_cert.jpg")  # Jméno z FRONT SHA!

# Výsledek:
# ab/abc123..._front.jpg  (originál front)
# ab/abc123..._back.jpg   (originál back, pojmenován podle front SHA)
# ab/abc123..._cert.jpg   (originál cert, pojmenován podle front SHA)
```

### Krok 5: Perspektivní korekce (warp) pro ML analýzu

**POZNÁMKA:** Warp se provede PRO ML analýzu (auto-detect v kroku 10), ale NEULOŽÍ se jako `_front.jpg`. Warp je pouze dočasný processing step.

Na základě detekovaného quadu systém provede perspektivní transformaci DOČASNĚ pro následnou embedding analýzu. ResNet50 model očekává známky v jednotném formátu 1300×1100 pixelů bez perspektivního zkreslení.

```python
# Warp JEN pro ML processing (NEULOŽIT!)
warped_temp = warp_image(original_front, quad, target_size=(1300, 1100))
# Tento warped_temp se použije pro auto-detect, pak se zahodí
```

### Krok 6: INSERT do reference_front

Systém vytvoří hlavní záznam v tabulce `reference_front` s následujícími hodnotami:
- `front_sha256` = vypočítaný SHA256 hash
- `stamp_type_id` = NULL (dosud neznámá denominace)
- `plate_id` = 0 (dosud neznámá tisková deska)
- `zp_no` = 0 (dosud neznámá pozice na archu)
- `confirmed` = 0 (pending, čeká na expert review po UC-02)

**Proč NULL pro stamp_type_id?** Protože uživatel ještě nepotvrdil klasifikaci (to se stane v UC-02). Systém sice provede auto-detect (krok 9), ale tento návrh není uložen do databáze - je pouze vrácen jako ephemeral response.

**Proč 0 pro plate_id a zp_no?** Konvence projektu: 0 znamená "neznámé/pending". NULL nelze použít kvůli NOT NULL constraint, protože i známky bez určené desky/pozice mohou být v GT (ověřené, ale nespecifikované).

### Krok 7: INSERT do user_uploads

Systém vytvoří záznam v tabulce `user_uploads`, který spojuje scan s uživatelem:
- `scan_id` = ID vytvořené v kroku 6
- `uploaded_by` = username@hostname formát (např. "milan@zenbook")
- `uploaded_at` = CURRENT_TIMESTAMP
- `upload_source` = 'web_ui' | 'api' | 'batch_import'

Tato tabulka umožňuje multi-user funkcionalitu - více uživatelů může mít ve svém view stejný sken (identifikovaný SHA256), ale každý vidí pouze své vlastní uploady.

### Krok 8: INSERT do inference_frames

Systém uloží YOLO frame jako cache do tabulky `inference_frames`:
- `scan_id` = ID z kroku 6
- `image_path` = NULL (není potřeba, máme SHA v reference_front)
- `x1, y1, x2, y2, x3, y3, x4, y4` = souřadnice quadu
- `bx, by, bw, bh` = axis-aligned bounding box
- `conf` = YOLO confidence
- `model_id` = ID YOLO modelu z `model_registry`

Cache slouží k tomu, aby při dalším použití skenu nebylo nutné znovu spouštět YOLO detekci.

### Krok 9: INSERT do scan_supplementary_files (pokud applicable)

Pokud uživatel nahrál back, cert nebo block soubory, systém pro každý vytvoří záznam v tabulce `scan_supplementary_files`:
- `scan_id` = ID z kroku 6
- `file_type` = 'back' | 'cert' | 'block'
- `file_sha256` = SHA256 konkrétního supplementary souboru (pro integritu)
- `uploaded_at` = CURRENT_TIMESTAMP

**UNIQUE constraint:** Jeden sken může mít maximálně jeden soubor každého typu (nelze nahrát dva certifikáty ke stejnému skenu).

**DŮLEŽITÉ o file naming:** Fyzické soubory v storage jsou pojmenované podle FRONT SHA (`{front_sha}_back.jpg`), ale v DB ukládáme jejich vlastní SHA hash pro integritu check. To umožňuje:
- Snadné načítání (všechny soubory pro jeden sken v jednom subdirectory)
- Integrity verification (můžeme zkontrolovat že back file nebyl modifikován)

### Krok 10: Auto-detect denominace (LIVE, bez DB storage)

Systém spustí auto-detection workflow, který se pokusí určit denominaci, tiskovou desku a pozici na archu. Tento proces používá tři fáze:
1. **Embedding similarity** - ResNet50 embeddings z value label crop (z DOČASNĚ warpovaného obrázku)
2. **OCR verification** - EasyOCR čtení číslic na hodnotovém štítku
3. **HSV color matching** - Kontrola barvy pro detekci reprintů/variant

**POZNÁMKA:** Pro embedding extraction se použije dočasný warp z kroku 5, který se po analýze zahodí. Originál zůstává uložený v storage.

Auto-detection vrátí TOP-5 kandidátů seřazených podle confidence score. **DŮLEŽITÉ:** Tyto výsledky NEJSOU uloženy do databáze, jsou pouze ephemeral response pro uživatele. Důvod: GT se může měnit (expert přidá nové scany), embeddings se přepočítávají (upgrade modelu), takže uložené výsledky by rychle zastaraly.

**Technické detaily:**
```python
# Konceptuální - existující funkce v denomination_utils.py
from common.denomination_utils import auto_detect_denomination

# Použije dočasný warp z kroku 5 pro analýzu
top5 = auto_detect_denomination(scan_id, warped_temp)
# Returns: [
#   {'stamp_type_id': 5, 'plate_id': 10, 'zp_no': 42, 'confidence': 0.95},
#   {'stamp_type_id': 5, 'plate_id': 10, 'zp_no': 43, 'confidence': 0.88},
#   ...
# ]
```

### Krok 11: Return response uživateli

Systém vrátí uživateli response s následujícími informacemi:
- `scan_id` - ID vytvořeného záznamu v reference_front
- `status` - 'new' (nový sken) nebo 'duplicate' (už existoval)
- `top5` - Pole TOP-5 kandidátů z auto-detect
- `message` - "Nový sken ve vašem view, zde jsou TOP-5 nejpodobnějších kandidátů"

**PRIVACY:** I když status interně může být 'duplicate' (sken nahrál jiný user), uživateli se VŽDY zobrazí "Nový sken ve vašem view" - nikdy se neodhalí existence jiných uživatelů.

---

## 🔑 KLÍČOVÉ KONCEPTY

### Koncept 1: SHA256 jako primární identifikátor

SHA256 hash originálního front obrázku slouží jako univerzální identifikátor skenu napříč systémem. Tento přístup umožňuje automatickou deduplikaci - pokud dva uživatelé nahrají fyzicky stejný sken (například stažený z aukčního webu), systém to pozná a neuloží soubory dvakrát.

Hash se počítá z **originálního uploadu**, NE z warpované verze. To znamená, že dva uživatelé kteří nahrají STEJNÝ originální soubor (byte-by-byte identický) dostanou STEJNÝ hash. Toto je žádoucí chování pro fungování deduplication - systém rozpozná že jde o duplicitní upload.

**Důsledky pro implementaci:**
- `reference_front.front_sha256` má UNIQUE constraint
- File path se NEUKLÁDÁ do DB (`file_path=NULL`), pouze SHA
- Path rekonstrukce: `f"/gt_data/scans/{sha[:2]}/{sha}_front.jpg"`
- Všechny související soubory (_back, _cert, _block) používají STEJNÝ SHA prefix

### Koncept 2: Multi-user view s privacy-first

Každý uživatel vidí pouze své vlastní uploady prostřednictvím tabulky `user_uploads`. Pokud UserA a UserB nahrají stejný sken (identický SHA256), vznikne:
- 1× záznam v `reference_front` (sdílený)
- 1× sada souborů v `/gt_data/scans/` (sdílená)
- 2× záznam v `user_uploads` (každý user má vlastní)

Oba uživatelé vidí sken jako "nový ve svém view" a systém NIKDY neodhaluje, že jej nahrál i někdo jiný. Toto zajišťuje privacy a zároveň efektivní využití storage (deduplikace souborů).

### Koncept 3: Pending stav bez embeddingů

Po UC-01 je sken ve stavu "pending" charakterizovaném:
- `stamp_type_id = NULL` - nezná se denominace
- `confirmed = 0` - neschváleno experty
- **Žádné embeddings v `reference_embeddings`** tabulce

Proč se nepočítají embeddingy hned? Protože systém potřebuje znát `stamp_type_id` → odvodit `drawing_id` → určit které crops extrahovat. Různé kresby známek (s popisem, s kroužky, abstraktní) mají různé sady cropů. Bez znalosti drawing_id nelze určit, co z obrázku vyříznout.

Embeddingy budou vypočítány až v UC-02 po potvrzení klasifikace uživatelem.

### Koncept 4: Originál storage vs warp pro ML

Front side obrázek je **uložen jako ORIGINÁL**, ne jako warp. Storage obsahuje přesně to co uživatel nahrál. Warp (perspektivní korekce na 1300×1100) je pouze dočasná transformace která se provede při ML analýze (auto-detect, embedding extraction), ale NEULOŽÍ se.

**Proč originál:**
- Warp můžeme kdykoli znovu vytvořit (máme quad v `inference_frames`)
- Originál po přepsání nelze obnovit
- Budoucí re-analýza s lepšími algoritmy vyžaduje originální data
- Rezoluce originálu je typicky vyšší (2400×3200) než warp (1300×1100)

Back side, certifikát a block také zůstávají v originální podobě protože slouží pro lidské posouzení experty (warp by degradoval čitelnost textu).

**Technický flow:**
```python
# 1. Uložit originál
shutil.copy(original_upload, f"{dir}/{sha}_front.jpg")

# 2. Pro ML analýzu: warp DOČASNĚ
warped_temp = warp_image(original, quad, (1300, 1100))

# 3. Použít warped_temp pro embeddings/OCR
embeddings = extract_embeddings(warped_temp)

# 4. Zahodit warped_temp (nepotřebujeme ho)
```

---

## ⚠️ DŮLEŽITÁ PRAVIDLA / BEST PRACTICES

### Pravidlo 1: VŽDY počítej SHA z originálního obrázku

SHA256 hash se MUSÍ počítat z originálního nahraného obrázku, NE z warpovaného. Důvod: dva uživatelé kteří nahrají STEJNÝ originál musí dostat STEJNÝ hash pro fungování deduplication. Warp má rounding errors při transformaci a dva warpy stejného originálu mohou mít mírně odlišné pixely → různé hashe → deduplication selhání.

**✅ Správně:**
```python
# 1. Uživatel nahraje original.jpg
original = load_image(upload_path)

# 2. SHA z ORIGINÁLU (PŘED warpem)
sha256 = compute_sha256(original)

# 3. Uložit ORIGINÁL pod SHA názvem
shutil.copy(upload_path, f"{target_dir}/{sha256}_front.jpg")

# 4. VOLITELNĚ: warp pro ML analýzu (dočasný)
warped_temp = warp_image(original, quad, (1300, 1100))
embeddings = extract_embeddings(warped_temp)
# warped_temp se zahodí
```

**❌ Špatně:**
```python
# ŠPATNĚ! Warp PŘED výpočtem SHA
warped = warp_image(original, quad, (1300, 1100))
sha256 = compute_sha256(warped)  # ŠPATNĚ! Deduplication nefunguje

# ŠPATNĚ! Ukládání warpu místo originálu
cv2.imwrite(f"{target_dir}/{sha256}_front.jpg", warped)  # ŠPATNĚ!
```

**Důsledky porušení:**
- ❌ Dva uživatelé s SAME originál dostanou různé SHA → duplicitní soubory v storage
- ❌ Ztráta originální rezoluce (warp je 1300×1100, originál 2400×3200)
- ❌ Nelze provést budoucí re-analýzu s lepšími algoritmy

### Pravidlo 2: stamp_type_id MUSÍ být NULL v UC-01

Nikdy nenastavuj `stamp_type_id` při INSERT v UC-01, i když máš výsledek z auto-detect. Důvod: auto-detect je pouze návrh pro uživatele, ne confirmed klasifikace. Až uživatel potvrdí v UC-02, teprve pak se `stamp_type_id` nastaví.

Porušení tohoto pravidla by způsobilo, že systém by počítal embeddingy s potenciálně špatným drawing_id, což by degradovalo kvalitu GT.

**✅ Správně:**
```python
db.execute("""
    INSERT INTO reference_front (front_sha256, stamp_type_id, confirmed)
    VALUES (?, NULL, 0)
""", (sha256,))
```

**❌ Špatně:**
```python
# Auto-detect vrátil stamp_type_id=5, ale NESMÍŠ ho uložit!
db.execute("""
    INSERT INTO reference_front (front_sha256, stamp_type_id, confirmed)
    VALUES (?, ?, 0)
""", (sha256, auto_detected_type_id))  # ŠPATNĚ!
```

### Pravidlo 3: Privacy - NIKDY neodhaluj existence jiných userů

Když UserB nahraje sken, který už UserA nahrál, systém interně pozná duplicitu (SHA existuje), ale UserB MUSÍ vidět response jako "Nový sken ve vašem view". NIKDY nesmí být zobrazen text jako:
- ❌ "Tento sken již nahrál UserA"
- ❌ "Duplikát existujícího skenu"
- ❌ "Již v databázi"

**✅ Správně:**
```python
# I když interně je to duplicate
if sha_exists_in_db(sha256):
    add_to_user_view(scan_id, current_user)
    return {
        'status': 'new_in_view',  # NE 'duplicate'!
        'message': 'Nový sken ve vašem view',
        'top5': run_auto_detect(scan_id)
    }
```

### Pravidlo 4: Auto-detect výsledky NEUKLÁDEJ do DB

TOP-5 kandidáti z auto-detect jsou ephemeral response - vrať je uživateli, ale NEUKLÁDEJ do DB. Důvod: GT se může změnit (expert přidá nové scany), embeddings se přepočítávají (model upgrade), uložené výsledky by rychle zastaraly a byly by misleading.

Pokud bys je uložil, musel bys je invalidovat při každé změně GT, což je maintenance overhead bez benefit.

**✅ Správně:**
```python
# Spočítej, vrať, ale NEUKLÁDEJ
top5 = auto_detect_denomination(scan_id)
return {'top5': top5}  # Ephemeral response
```

**❌ Špatně:**
```python
# NEUKLÁDEJ do DB!
db.execute("""
    INSERT INTO auto_detect_results (scan_id, top5_json)
    VALUES (?, ?)
""", (scan_id, json.dumps(top5)))  # ŠPATNĚ!
```

---

## 🔧 TECHNICKÁ REFERENCE

### Databázové tabulky (čtení)

UC-01 provádí INSERT do následujících tabulek (detaily viz DB_STRUKTURA_PRUHLEDCE.md):

**reference_front:**
```sql
CREATE TABLE reference_front (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    front_sha256 TEXT NOT NULL UNIQUE,
    stamp_type_id INTEGER,  -- NULL v UC-01
    plate_id INTEGER NOT NULL DEFAULT 0,
    zp_no INTEGER NOT NULL DEFAULT 0,
    confirmed INTEGER NOT NULL DEFAULT 0,
    suspect_flag INTEGER DEFAULT 0,
    suspect_reason TEXT,
    notes TEXT,
    added_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

**user_uploads:**
```sql
CREATE TABLE user_uploads (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    scan_id INTEGER NOT NULL,
    uploaded_by TEXT NOT NULL,
    uploaded_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    upload_source TEXT,
    UNIQUE (scan_id, uploaded_by),
    FOREIGN KEY (scan_id) REFERENCES reference_front(id)
);
```

**scan_supplementary_files:**
```sql
CREATE TABLE scan_supplementary_files (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    scan_id INTEGER NOT NULL,
    file_type TEXT NOT NULL,  -- 'back', 'cert', 'block'
    file_sha256 TEXT NOT NULL,
    uploaded_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (scan_id, file_type),
    FOREIGN KEY (scan_id) REFERENCES reference_front(id)
);
```

### API (konceptuální - bude implementováno)

```python
def process_user_upload(
    front_file: Union[str, BinaryIO],
    back_file: Optional[Union[str, BinaryIO]] = None,
    cert_file: Optional[Union[str, BinaryIO]] = None,
    block_file: Optional[Union[str, BinaryIO]] = None,
    uploaded_by: str,
    upload_source: str = 'web_ui'
) -> dict:
    """
    Zpracuje upload známky od uživatele (UC-01).
    
    Args:
        front_file: Líc známky (povinný)
        back_file: Rub známky (volitelný)
        cert_file: Certifikát (volitelný)
        block_file: Bloková čtveřice (volitelný)
        uploaded_by: Username ve formátu "user@hostname"
        upload_source: Zdroj uploadu
        
    Returns:
        {
            'scan_id': int,
            'status': 'new' | 'new_in_view',
            'top5': List[Dict],  # Kandidáti z auto-detect
            'message': str
        }
        
    Raises:
        YOLODetectionError: Pokud YOLO nenajde rámeček
        FileStorageError: Pokud selže uložení souborů
    """
    # Funkce bude implementována
    pass
```

### File storage path reconstruction

```python
def get_scan_path(sha256: str, file_type: str, env: str) -> Path:
    """
    Rekonstruuje cestu k souboru ze SHA256.
    
    Args:
        sha256: SHA256 hash souboru
        file_type: 'front', 'back', 'cert', 'block'
        env: 'dev', 'prod', 'sandbox'
        
    Returns:
        Path: Absolutní cesta k souboru
        
    Example:
        >>> path = get_scan_path('abcd1234...', 'front', 'dev')
        >>> # ~/ProjektHradcany/dev/data/gt/scans/ab/abcd1234..._front.jpg
    """
    # Implementováno v gt_upload_utils.py
    pass
```

---

## 🐛 ZNÁMÉ PROBLÉMY / OMEZENÍ

### Problém 1: YOLO selhání při špatné kvalitě skenu

YOLO model může selhat při detekci rámečku, pokud je sken příliš rozmazaný, má extrémní osvětlení nebo známka je částečně zakrytá. V takovém případě workflow končí chybou a uživatel musí nahrát kvalitnější sken.

**Workaround:** Systém by mohl nabídnout manuální výběr rámečku (uživatel klikne na 4 rohy), ale toto není zatím implementováno. Pro GT kvalitu je lepší vyžadovat kvalitní skeny než akceptovat špatné s manuální korekcí.

### Problém 2: Auto-detect není 100% přesný

Auto-detection může vrátit nesprávný návrh denominace, zejména pro:
- 1000h známky (OCR čte "100" místo "1000")
- 15h TD4 (kvalita reference dat je horší)
- Reprints a varianty (embedding podobnost vysoká, ale jiná tisková deska)

**Důsledek:** Uživatel může být zmaten, pokud TOP-1 kandidát není správný. Řešení: UC-02 umožňuje uživateli opravit klasifikaci nebo vybrat z TOP-5.

### Problém 3: SHA collision risk (teoretický)

Teoreticky existuje riziko SHA256 kolize (dva různé obrázky se stejným hashem), ale pravděpodobnost je astronomicky nízká (2^-256). Pro praktické účely je toto riziko zanedbatelné.

Pokud by kolize nastala, systém by přepsal existující soubor novým, což by degradovalo GT. Řešení by vyžadovalo dodatečný fallback mechanismus (např. MD5 jako sekundární hash), ale to není implementováno.

---

## 🔗 SOUVISLOSTI

Tento Use Case je vstupním bodem do GT Management systému a přímo navazuje na následující workflows:

**Navazující Use Cases:**
- [UC-02: User Confirm Classification](./UC-02_User_Confirm_Classification.md) - Potvrzení klasifikace (NÁSLEDUJE VŽDY po UC-01)
- [UC-03: User View My Scans](./UC-03_User_View_My_Scans.md) - Prohlížení nahraných skenů
- [UC-17: User Delete From View](./UC-17_User_Delete_From_View.md) - Smazání pending skenu

**Používá komponenty:**
- `common/yolo_utils.py` - YOLO detekce rámečku
- `common/denomination_utils.py` - Auto-detection denominace
- `common/db_utils.py` - Databázové operace (bude rozšířeno)
- `dev/code/recognize_stamp.py` v3.2.1 - Partial implementation (warp, embedding)

**Související dokumenty:**
- [decisions/gt_management_use_cases.md](../decisions/gt_management_use_cases.md) - Celkový koncept GT Management
- [decisions/sha256_storage_strategy.md](../decisions/sha256_storage_strategy.md) - Proč SHA256 storage
- [base/file_storage_architecture.md](./file_storage_architecture.md) - Detaily file storage
- [base/pipeline_recognition.md](./pipeline_recognition.md) - Recognition pipeline (Fáze 2: YOLO + warp)
- [decisions/schema_v3_1_0_nullable_columns.md](../decisions/schema_v3_1_0_nullable_columns.md) - Proč nullable stamp_type_id

**Implementováno (částečně) v:**
- `dev/code/recognize_stamp.py` v3.2.1 - Obsahuje YOLO detection, warp, embedding extraction
- Kompletní UC-01 workflow bude implementován jako nová funkce v `common/gt_utils.py` nebo `common/upload_utils.py` (TBD)

---