# Recognition Pipeline - Projekt Hradčany

**Tags:** `#pipeline` `#recognition` `#zlatý-standard`  
**Verze:** 2.0.1  
**Datum:** 2026-01-14  
**Status:** Active - Production Reference

---

## 🎯 ÚČEL

Recognition pipeline je hlavní součást systému Hradčany, která automaticky identifikuje poštovní známky ze skenů nebo fotografií. Systém přijímá jako vstup obrázek známky (sken nebo fotografie) a vrací seznam nejpodobnějších kandidátů z referenční databáze Ground Truth. Každý kandidát je přesně určen kombinací tiskové desky (TD) a známkového pole (ZP), společně se statistikami podobnosti.

Pipeline řeší základní problém variability vstupních dat. Uživatelské skeny se liší v kvalitě, rozlišení, natočení a perspektivě. Známka může být vyfocena z úhlu, mít různé osvětlení nebo být částečně zakryta. Pipeline musí tyto variace normalizovat do jednotného formátu, ze kterého lze spolehlivě extrahovat charakteristické rysy pro porovnání s referenčními známkami.

Celý proces je navržen jako deterministický - stejný vstup vždy produkuje stejný výstup. To je kritické pro reprodukovatelnost výsledků a možnost systematického testování přesnosti systému. Deterministické chování zajišťujeme fixním seedem pro projekční hlavu embedding modelu, konstantními parametry pro warp transformaci a L2 normalizací všech embedding vektorů.

Tento dokument popisuje zlatý standard recognition pipeline tak, jak je implementován v produkčním programu `recognize_stamp.py` verze 3.2.0. Je to referenční dokumentace pro všechny části systému které potřebují rozpoznávat známky. Nové implementace (například webové API nebo batch processing nástroje) musí dodržet workflow popsané v tomto dokumentu aby zajistily kompatibilitu výsledků.

---

## 📋 PŘEHLED

Recognition pipeline se skládá ze čtrnácti fází které na sebe lineárně navazují. Prvních devět fází tvoří jádro rozpoznávacího procesu - jsou povinné a musí být dodrženy ve všech implementacích. Fáze deset až čtrnáct jsou volitelné pomocné funkce specifické pro jednotlivé use cases.

**Core recognition workflow (fáze 1-9):**
Tyto fáze implementují samotný rozpoznávací algoritmus. Začínáme načtením konfigurace z databáze, inicializací embedding modelu a detekcí rámečku známky pomocí YOLO. Následuje perspektivní transformace (warp) známky do standardizovaného formátu 1300×1100 pixelů. Pokud uživatel nezadal typ známky (denomination), systém ho automaticky detekuje pomocí multi-phase algoritmu který kombinuje embedding similarity, OCR a HSV color matching. Po určení typu známky načteme definice výřezů (crops) specifické pro danou kresbu, vyřízneme tyto oblasti a vypočítáme pro každou 512-dimenzionální embedding vektor. Tyto query embeddings pak porovnáme s referenčními embeddingy v databázi pomocí cosine similarity, agregujeme výsledky po kandidátech (TD + ZP kombinace) a seřadíme je podle průměrné podobnosti.

**Volitelné pomocné funkce (fáze 10-14):**
Tyto fáze nejsou součástí zlatého standardu pro recognition. Slouží pro specifické use cases programu `recognize_stamp.py` - například vytvoření záznamu v databázi (fáze 10), načtení poznámek z .notes.json souboru (fáze 11), výpis výsledků (fáze 12), validace proti očekávanému výsledku při testování (fáze 13) a vytvoření vizuální mozaiky pro kontrolu (fáze 14). Tyto fáze můžou být v jiných implementacích vynechány nebo nahrazeny podle potřeby.

Klíčovou charakteristikou pipeline je hybrid přístup k detekci rámečku (fáze 3) - systém nejdřív zkusí načíst již detekovaný quad z databázové cache, což je velmi rychlé (řádově milisekundy). Teprve pokud quad v cache není, spustí se YOLO inference která je podstatně pomalejší (řádově stovky milisekund). Tento přístup výrazně zrychluje opakované zpracování stejných skenů, což je běžný scénář při testování a ladění systému.

---

## 🔄 HLAVNÍ WORKFLOW

### Fáze 1: Načtení konfigurace z databáze

V první fázi pipeline načítá z databáze základní konfiguraci potřebnou pro rozpoznávání. Konkrétně jde o informace o YOLO modelu použitém pro detekci rámečků a o embedding modelu včetně projekční hlavy. Tyto informace jsou uloženy v tabulkách `model_registry` a `model_embed_head`.

Načtení konfigurace z databáze místo hardcoded hodnot v kódu umožňuje verzování modelů. Když trénujeme nový YOLO model nebo měníme architekturu embedding sítě, stačí přidat nový záznam do databáze a všechny běžící instance pipeline automaticky použijí novou verzi. To je kritické pro konzistenci výsledků - všechny embeddings v databázi musí být vypočítané stejným modelem se stejnou projekční hlavou.

Databázový záznam obsahuje SHA256 hash souboru s modelem, což umožňuje verifikovat že používáme správnou verzi. Pro embedding model je navíc uložen seed použitý pro inicializaci projekční hlavy (typicky 12345). Tento seed musí být identický při výpočtu referenčních embeddingů i query embeddingů, jinak by nebyly embeddings porovnatelné.

**Technické detaily:**
```sql
-- Načtení informací o YOLO modelu
SELECT id, tag, sha256 FROM model_registry WHERE id = 1;

-- Načtení informací o embedding modelu včetně projekční hlavy
SELECT m.id, m.tag, m.sha256, h.seed, h.head_sha256
FROM model_registry m
JOIN model_embed_head h ON m.id = h.model_id
WHERE m.id = 3;
```

### Fáze 2: Inicializace embedding modelu

Druhá fáze inicializuje embedding model který bude použit pro výpočet query embeddingů. Konkrétně jde o RealEncoder - wrapper kolem ResNet50 který má zabudované preprocessing (konverze BGR→RGB, resize na 224×224, normalizace do rozsahu [0,1]) a deterministickou projekční hlavu pro redukci dimenze z 2048D na 512D.

RealEncoder je kritická komponenta protože musí být naprosto identický s tím který byl použit pro výpočet referenčních embeddingů uložených v databázi. Jakýkoliv rozdíl v preprocessing (například jiná normalizace) nebo v projekční hlavě (jiný seed) by vedl k neporovnatelným embeddingům a recognition by selhal.

Projekční hlava je inicializována z .pth souboru který obsahuje natrénované váhy neuronové sítě. Před inicializací se nastaví fixní seed (typicky 12345) aby byla hlava deterministická - stejný vstup vždy produkuje stejný výstup. Toto je důležité protože projekční hlava obsahuje random inicializaci některých vrstev a bez fixního seedu by každá instance modelu měla jiné váhy.

Po inicializaci se model přepne do evaluation módu (`model.eval()`) aby se vypnuly vrstvy které se chovají jinak během trénování (například dropout nebo batch normalization). Všechny výpočty probíhají s `torch.no_grad()` aby se nealokovala paměť pro gradienty - při inference je nepotřebujeme.

**Technické detaily:**
```python
from common.embedding_utils import RealEncoder

# Inicializace RealEncoder s deterministickou hlavou
encoder = RealEncoder(
    model_path=model_file,        # ResNet50 weights
    head_path=head_file,          # Projekční hlava .pth
    seed=12345,                   # Fixní seed pro determinismus
    device='cpu'                  # CPU inference (GPU není nutné)
)

# Model je automaticky v eval() módu
# Všechny výpočty probíhají v torch.no_grad() context
```

### Fáze 3: Detekce rámečku známky (YOLO hybrid approach)

Třetí fáze detekuje čtyři rohové body rámečku známky v původním obrázku. Tento quad (čtveřice bodů) je potřebný pro perspektivní transformaci v následující fázi. Detekce využívá hybrid přístup který kombinuje výhody rychlé cache s robustností YOLO inference.

**Workflow detekce:**

První krok je pokus o načtení quad z databázové cache (tabulka `inference_frames`). Pokud byl tento konkrétní sken už někdy v minulosti zpracován, máme uložený detekovaný quad a můžeme ho okamžitě použít. Načtení z cache je extrémně rychlé (řádově 1-2 milisekundy) protože jde pouze o SELECT query z SQLite databáze.

Pokud quad v cache není (cache miss), spustí se YOLO inference. YOLO model je natrénovaný na detekci rámečků československých známek a vrací oriented bounding box (OBB) - tedy čtyři rohové body místo klasického axis-aligned obdélníku. YOLO inference je podstatně pomalejší než cache lookup (řádově 300-500 milisekund na CPU), ale stále dostatečně rychlá pro interaktivní použití.

Pokud YOLO detekce uspěje (confidence score je nad threshold, typicky 0.95), výsledný quad se uloží do databázové cache pro příští použití. Toto uložení probíhá asynchronně - pokud selže (například kvůli chybě databáze), YOLO výsledek je stále platný a pipeline pokračuje normálně. Cachování je optimalizace, ne požadavek pro správnou funkci.

**Proč hybrid přístup:**
Opakované zpracování stejných skenů je běžný scénář při vývoji a testování systému. Například při ladění threshold parametrů nebo testování nové verze embedding modelu musíme zpracovat celý testovací dataset (6800 skenů). Bez cache by každý běh trval přes hodinu pouze na YOLO detekci. S cache je druhý a další běh stokrát rychlejší protože quad je už uložený.

**Technické detaily:**
```python
# Pokus o načtení z cache
frame_row = get_cached_frame_by_path(conn, input_path)

if frame_row is not None:
    # Cache hit - použij uložený quad
    quad = parse_quad_from_frame_row(frame_row)
    source = 'cache'
else:
    # Cache miss - spusť YOLO inference
    quad, conf, bbox = detect_frame_yolo(
        image_path=input_path,
        model_path=yolo_model_path,
        conf_threshold=0.95
    )
    
    # Pokud YOLO úspěšné, cachuj výsledek
    if quad is not None:
        upsert_inference_frame(conn, input_path, quad, conf, bbox)
        source = 'yolo'
```

**Formát quad:**
Quad je numpy array tvaru (4, 2) obsahující souřadnice čtyř rohových bodů v pořadí: Top-Left, Top-Right, Bottom-Right, Bottom-Left. Souřadnice jsou v pixelech původního obrázku (např. 2400×3000). Toto pořadí je kritické pro korektní warp v další fázi.

### Fáze 4: Warp a normalizace na 1300×1100

Čtvrtá fáze provádí perspektivní transformaci (warp) známky do standardizovaného formátu 1300×1100 pixelů. Vstupem je původní sken v libovolném rozlišení a quad detekovaný v předchozí fázi. Výstupem je frontálně narovnaná známka v jednotném rozlišení, připravená pro extrakci výřezů.

**Účel warp transformace:**
Uživatelské skeny jsou různě natočené, vyfocené z úhlu, nebo mají různé rozlišení. Některé skeny jsou 2400×3000 pixelů z profesionálního skeneru, jiné 1920×1080 z mobilního telefonu. Známka může být na skeneru položená šikmo nebo mírně zkosená. Warp transformace všechny tyto variace normalizuje do jednotného formátu kde známka je vždy frontálně narovnaná, má stejné rozlišení a stejné proporce.

Tato normalizace je kritická pro následující fáze. Definice výřezů (crops) v databázi jsou uložené jako relativní souřadnice na normalizované známce 1300×1100. Například "hodnotový štítek" je definovaný jako elipsa na pozici (50%, 86%) se specifickým poloměrem. Tyto relativní souřadnice funují pouze když je známka ve standardizovaném formátu.

**Jak warp funguje:**
Z quad (čtyři rohové body) a cílového rozměru 1300×1100 se vypočítá homografická matice 3×3 která definuje perspektivní transformaci. Tato matice se aplikuje na původní obrázek pomocí OpenCV funkce `cv2.warpPerspective()`. Výsledkem je obdélníkový obrázek 1300×1100 kde známka je narovnaná a má standardizované proporce.

Rozměr 1300×1100 byl zvolen empiricky jako kompromis mezi kvalitou detailů a výpočetní náročností. Je dostatečně velký aby zachoval jemné detaily jako spirály nebo číslovky na hodnotovém štítku, ale zároveň dostatečně malý aby výpočet embeddingů byl rychlý. Proporce 1300:1100 přibližně odpovídají reálným proporcím československých známek série Hradčany.

**Fallback při selhání detekce:**
Pokud quad nebyl v předchozí fázi detekován (YOLO selhalo), použije se fallback - prostý resize původního obrázku na 1300×1100. To není ideální protože známka může být zkosená nebo mít špatné proporce, ale umožňuje to pokračovat v pipeline alespoň s přibližným výsledkem. V praxi YOLO detekce selhává velmi vzácně (< 0.1% případů) takže fallback je emergency řešení. (MB ToDo: ukončit pipeline, pokud není YOLO detekce úspěšná)

**Technické detaily:**
```python
# Načtení původního obrázku
orig_bgr = cv2.imread(input_path)

if quad is not None:
    # Warp pomocí homografie
    H = compute_homography_matrix(quad, target_size=(1300, 1100))
    full_norm = cv2.warpPerspective(orig_bgr, H, (1300, 1100))
else:
    # Fallback - prostý resize
    full_norm = cv2.resize(orig_bgr, (1300, 1100), interpolation=cv2.INTER_AREA)
```

**Výstup:**
Proměnná `full_norm` obsahuje normalizovanou známku jako numpy array tvaru (1100, 1300, 3) v BGR color space. Tento obrázek je použit ve všech následujících fázích pro extrakci výřezů.

### Fáze 5: Auto-detect denomination (pokud typ není zadán)

Pátá fáze automaticky detekuje typ známky (denomination) pokud ho uživatel nezadal pomocí parametru `--denomination`. Detekce používá multi-phase algoritmus který kombinuje embedding similarity, OCR a HSV color matching pro robustní klasifikaci.

**Kdy se fáze spouští:**
Pokud uživatel zadal `--denomination` parametr (např. `--denomination 500h`), tato fáze se přeskakuje a použije se zadaný typ. To je backward compatibility režim který umožňuje manuální zadání typu když víme co hledáme. Pokud parametr není zadán, spustí se auto-detect workflow.

**Multi-phase detekce:**

První fáze je embedding-based detection. Systém vyřízne hodnotový štítek z normalizované známky (oblast kde je vytištěná hodnota - např. "500"), vypočítá embedding tohoto výřezu a porovná ho s embeddingy hodnotových štítků všech 26 typů známek v databázi. Typicky se TOP-3 kandidáti liší jen o desetiny procenta v similarity, takže pouze embedding není dostatečný pro jednoznačné určení.

Druhá fáze je OCR detekce (disambiguation). Pro TOP-K kandidátů z embedding fáze (typicky K=10) se spustí OCR na hodnotovém štítku. OCR model (EasyOCR) extrahuje text ze štítku a pokusí se ho rozeznat jako číslo. Pokud OCR přečte například "500", můžeme s vysokou jistotou říct že jde o 500h známku i když embedding similarity byla podobná jako u 50h15 nebo 50h16.

Třetí fáze je porovnání barev (HSV color verification). Některé typy známek existují v různých barevných variantách (např. 50h15 modrá vs 50h16 zelená) které mají velmi podobné embeddings ale liší se barvou. Pro finální disambiguaci se porovná průměrná barva známky v HSV color space s definovanými HSV pravidly pro každý typ. Pokud barva nesedí, snižuje se confidence nebo se kandidát zamítne.

**Výstup:**
Auto-detect vrací slovník s klíči: `denomination` (např. "500h"), `stamp_type_id` (ID v databázi), `confidence_level` (HIGH/MEDIUM/LOW/MANUAL_REVIEW), `similarity` (embedding score), `ocr_text` (rozpoznaný text), `hsv_match` (boolean). Pokud je confidence LOW nebo MANUAL_REVIEW, pipeline se zastaví s chybovou hláškou protože nemůžeme pokračovat s nejistou klasifikací.

**Proč multi-phase:**
Embedding alone dává dobré výsledky pro většinu případů ale selhává u velmi podobných typů (10h5 vs 15h, 50h15 vs 50h16). OCR alone selhává u poškozených nebo špatně vytištěných štítků (např. 1000h často čte jako "100"). Kombinace všech tří metod dosahuje 96.9% přesnosti na testovacím datasetu 6800 skenů.

**Technické detaily:**
```python
from common.denomination_utils import auto_detect_denomination

result = auto_detect_denomination(
    conn=conn,
    warped_bgr=full_norm,
    encoder=encoder,
    topk=10,
    embedding_threshold=0.85,
    debug=False
)

if result['confidence_level'] in ['LOW', 'MANUAL_REVIEW']:
    raise ValueError(f"Nízká confidence: {result['confidence_level']}")

denomination = result['denomination']  # např. "500h"
stamp_type_id = result['stamp_type_id']  # např. 5
```

### Fáze 6: Načtení definic výřezů pro danou kresbu

Šestá fáze načítá z databáze definice výřezů (crops) pro konkrétní kresbu známky. Československé známky série Hradčany existují v šesti různých kresbách (s popisem, s kroužky, abstraktní, atd.) a každá kresba má svoje specifické výřezy které obsahují charakteristické rysy potřebné pro identifikaci.

**Co je crop:**
Crop je definovaná oblast na normalizované známce 1300×1100 která obsahuje nějaký charakteristický rys - například spirála vlevo, obloha vpravo nahoře, keř s větvemi, nebo hodnotový štítek. Každý crop má přesně definovaný tvar (obdélník, kruh, elipsa nebo polygon), pozici (relativní souřadnice od 0.0 do 1.0), padding (okraj navíc kolem definované oblasti) a polaritu (tmavé na světlém nebo světlé na tmavém).

Tyto definice jsou uložené v databázové tabulce `drawing_crops` a jsou pevně spojené s konkrétní kresbou. Například kresba "s popisem" má crop na pozici nápisu POŠTA, zatímco kresba "s kroužky" má crop na pozici horních kroužků. Crops jsou navržené expertně aby obsahovaly oblasti které se liší mezi různými tiskovými deskami nebo známkovými poli.

**Proč multiple crops:**
Porovnávání celé známky jako jediného embeddings není dostatečné pro jemné rozlišení mezi deskami a pozicemi. Různé části známky obsahují různé typy informací - spirála rozlišuje známkové pole v rámci desky, obloha má texturu specifickou pro pole, keř s větvemi rozlišuje desku. Porovnáváním každého cropu samostatně dostáváme vector of similarities místo single scalar, což umožňuje jemnější matching.

V databázi jsou pro každý crop uložené parametry jako `is_required` (tento crop MUSÍ být použit pro matching) a `verified` (tento crop byl ověřen expertem). Pipeline načítá pouze required verified crops - typicky 4-8 crops per drawing.

**Technické detaily:**
```sql
-- Načtení required crops pro kresbu ID=5 (500h)
SELECT id, code, name, crop_type, grid_rect_json, padding, polarity
FROM drawing_crops
WHERE drawing_id = 5 AND is_required = 1 AND verified = 1
ORDER BY id;
```

**Výstup:**
Seznam CropDef objektů obsahujících: `id` (unique crop ID), `code` (např. "spirala_levy"), `crop_type` (box/circle/ellipse/poly), `grid_rect_json` (relativní souřadnice), `padding` (např. 0.05), `polarity` (ink/paper). Tyto definice jsou použité v další fázi pro extrakci konkrétních oblastí z normalizované známky.

### Fáze 7: Extrakce výřezů a výpočet embeddingů

Sedmá fáze je jádro rozpoznávacího procesu. Pro každý crop definovaný v předchozí fázi se z normalizované známky vyřízne konkrétní oblast, tato oblast se preprocessing upraví do formátu očekávaného embedding modelem a vypočítá se 512-dimenzionální embedding vektor. Výsledkem jsou query embeddings které reprezentují charakteristické rysy analyzované známky.

**Extrakce cropu:**
Každý crop má definovaný tvar (box/circle/ellipse/poly) a relativní souřadnice na normalizované známce 1300×1100. Tyto relativní souřadnice se převedou na pixelové souřadnice a z `full_norm` obrázku se vyřízne příslušná oblast. Pro non-rectangular shapes (kruh, elipsa) se vytvoří maska která nastaví pixely mimo tvar na černou.

Například crop "spirála vlevo" je definovaný jako elipsa na pozici (cx=0.25, cy=0.30) s poloměry (rx=0.08, ry=0.10). Na normalizované známce 1300×1100 to odpovídá středu na (325, 330) pixelech s poloměry (104, 110) pixelů. Z tohoto regionu se vyřízne obdélník a pixely mimo elipsu se nastaví na nulu.

K výřezu se přidá padding (typicky 5% navíc kolem definované oblasti) aby se zachytily i okolní pixely které mohou obsahovat užitečnou informaci. Polarity parameter (ink/paper) určuje jestli invertovat pixely - "ink" znamená tmavé na světlém (normální), "paper" znamená světlé na tmavém (invertované).

**Výpočet embedding:**
Každý vyříznutý crop se předá do RealEncoder který ho převede na 512D vektor. RealEncoder interně provádí:
1. BGR → RGB konverzi (OpenCV používá BGR, PyTorch očekává RGB)
2. Resize na 224×224 (vstupní rozměr ResNet50)
3. Normalizaci do rozsahu [0, 1] (dělení 255.0)
4. Konverzi na PyTorch tensor
5. Forward pass ResNet50 → 2048D features
6. Projekce 2048D → 512D přes natrénovanou hlavu
7. L2 normalizaci výsledného vektoru (norma = 1.0)

L2 normalizace je kritická protože umožňuje použít dot product místo cosine similarity v další fázi. Pro dva L2-normalizované vektory platí: cosine_similarity(a, b) = dot(a, b). Dot product je výrazně rychlejší než výpočet cosine similarity.

**Technické detaily:**
```python
query_embeddings = []

for crop_def in required_crops:
    # 1. Extrahuj crop z full_norm podle definice
    crop_img = extract_crop_region(
        full_norm, 
        crop_def.crop_type,
        crop_def.grid_rect_json,
        crop_def.padding,
        crop_def.polarity
    )
    
    # 2. Vypočítaj embedding
    with torch.no_grad():
        emb = encoder(crop_img)  # (512,) L2-normalized
    
    query_embeddings.append({
        'crop_id': crop_def.id,
        'crop_code': crop_def.code,
        'embedding': emb.cpu().numpy()
    })
```

**Výstup:**
Seznam slovníků kde každý obsahuje `crop_id`, `crop_code` a `embedding` (numpy array shape (512,), dtype float32, L2-normalized). Počet embeddingů odpovídá počtu required crops (typicky 4-8).

### Fáze 8: Matching s referenčními embeddingy

Osmá fáze porovnává query embeddings vypočítané v předchozí fázi s referenčními embeddingy uloženými v databázi. Pro každý query embedding najdeme TOP-K nejpodobnějších referenčních embeddingů pomocí cosine similarity. Výsledkem je seznam match candidates s jejich similarity scores.

**Referenční embeddings v databázi:**
Databázová tabulka `reference_embeddings` obsahuje embeddings vypočítané ze všech známých referenčních skenů. Pro každý referenční sken (například 500h TD1 ZP42) existuje několik embeddingů - jeden pro každý required crop. Celkem databáze obsahuje přibližně 50,000 embeddingů (6800 skenů × průměrně 7 crops per sken).

Každý embedding je uložený jako BLOB (binary large object) - serializovaný numpy array float32[512] který zabírá 2048 bytů. K embeddings jsou navázané metadata: `scan_id` (odkaz na referenční sken), `crop_id` (který crop to je), `model_id` (kterým modelem byl vypočítaný), `l2_norm` (norma vektoru, vždy 1.0).

**Matching algoritmus:**
Pro každý query embedding se provede SQL query která najde TOP-K nejpodobnějších referenčních embeddingů se STEJNÝM crop_id. Proč stejný crop_id? Protože embeddings různých crops nejsou porovnatelné - spirála má jiný význam než obloha, jejich embeddings leží v jiných regionech 512D prostoru. Porovnáváme vždy pouze "spirála query" vs "spirály reference".

Podobnost se počítá jako dot product (skalární součin) dvou L2-normalizovaných vektorů. Pro vektory s normou 1.0 platí: dot(a, b) = cosine(angle(a, b)). Čím větší dot product, tím menší úhel mezi vektory, tím podobnější jsou embeddings. Hodnoty jsou v rozsahu [-1, 1] kde 1 = identické, 0 = ortogonální, -1 = opačné.

Matching se provádí pouze v rámci stejné kresby (drawing_id). Nelze porovnávat známku s kresbou "s popisem" proti známce s kresbou "abstraktní" protože mají jiné crops a embeddings nejsou kompatibilní.

**Technické detaily:**
```sql
-- Pro každý query embedding (např. spirala_levy, crop_id=8)
-- najdi TOP-10 nejpodobnějších referenčních embeddingů

WITH query AS (
    SELECT ? AS query_embedding  -- binding: serialized numpy array
)
SELECT 
    re.scan_id,
    rf.plate_id,
    rf.zp_no,
    dot_product(re.vec, query.query_embedding) AS similarity
FROM reference_embeddings re
JOIN reference_front rf ON re.scan_id = rf.id
CROSS JOIN query
WHERE 
    re.crop_id = 8                    -- STEJNÝ crop jako query
    AND rf.drawing_id = 5             -- STEJNÁ kresba
    AND rf.confirmed = 1              -- pouze potvrzené reference
ORDER BY similarity DESC
LIMIT 10;
```

**Výstup:**
Pro každý query embedding dostaneme seznam TOP-K matches obsahující: `scan_id` (referenční sken), `plate_id` (tisková deska), `zp_no` (známkové pole), `similarity` (cosine similarity score 0.0-1.0). Celkem tedy máme N_crops × K matches (např. 5 crops × 10 matches = 50 match records).

### Fáze 9: Agregace a ranking kandidátů

Devátá fáze agreguje match results z předchozí fáze do kandidátů definovaných kombinací (tisková deska, známkové pole) a seřadí je podle průměrné podobnosti. Výsledkem je ranked list TOP-K kandidátů kde každý kandidát má agregované statistiky podobnosti across all crops.

**Co je kandidát:**
Kandidát je kombinace (plate_id, zp_no) - například TD1 ZP42. Pro každého kandidáta máme několik similarity scores - jeden pro každý crop který byl matchován. Například pro kandidáta TD1 ZP42 můžeme mít: spirala_levy=0.89, obloha=0.87, ker_vetve=0.91, hodnotovy_stitek=0.93.

**Agregace:**
Pro každého kandidáta vypočítáme agregační statistiky ze všech similarity scores:
- **mean** (průměr): hlavní metrika pro ranking, reprezentuje celkovou podobnost
- **median** (medián): robustnější vůči outliers než mean
- **min** (minimum): ukazuje nejhorší match, užitečné pro detekci false positives
- **max** (maximum): ukazuje nejlepší match
- **std** (směrodatná odchylka): ukazuje variabilitu matchů
- **n** (count): počet cropů které byly matchované

Průměrná podobnost (mean) je použitá jako primární ranking metrika. Kandidáti jsou seřazení sestupně podle mean similarity - čím vyšší průměr, tím podobnější kandidát.

**Proč agregace:**
Single crop match může být zavádějící - například spirála může být podobná náhodou i když zbytek známky je úplně jiný. Agregací across multiple crops dostáváme robustnější klasifikaci. Pokud má kandidát konzistentně vysokou podobnost na všech crops (mean=0.92, median=0.91, min=0.89, max=0.94), je to silný signál že jde o správný match. Naopak velká variance (min=0.75, max=0.95) naznačuje nejistý match.

**Best scan selection:**
Pro každého kandidáta (TD, ZP kombinaci) může existovat několik referenčních skenů v databázi (různé exempláře stejné známky). Vybírá se "best scan" - ten se nejvyšší průměrnou similarity. Tento scan se použije pro vizualizaci v případě že uživatel požaduje compare-full mozaiku.

**Technické detaily:**
```python
from collections import defaultdict
import numpy as np

# Agregace matchů po kandidátech
candidates = defaultdict(list)

for match in all_matches:
    key = (match['plate_id'], match['zp_no'])
    candidates[key].append(match['similarity'])

# Výpočet statistik pro každého kandidáta
ranked_candidates = []

for (plate_id, zp_no), similarities in candidates.items():
    sim_array = np.array(similarities)
    
    candidate = {
        'plate_id': plate_id,
        'zp_no': zp_no,
        'mean': np.mean(sim_array),
        'median': np.median(sim_array),
        'min': np.min(sim_array),
        'max': np.max(sim_array),
        'std': np.std(sim_array),
        'n': len(sim_array)
    }
    
    ranked_candidates.append(candidate)

# Seřazení sestupně podle mean similarity
ranked_candidates.sort(key=lambda c: c['mean'], reverse=True)

# TOP-K výběr (typicky K=10)
top_candidates = ranked_candidates[:args.topk]
```

**Výstup:**
Seznam TOP-K kandidátů (defaultně K=10) seřazených podle průměrné podobnosti. Každý kandidát obsahuje: `plate_id`, `zp_no`, `denomination`, `color_name`, agregační statistiky (`mean`, `median`, `min`, `max`, `std`, `n`), `best_scan_id` (pro vizualizaci). Tímto končí core recognition workflow - fáze 1-9.

### Fáze 10: DB UPSERT - Vytvoření/aktualizace záznamu (VOLITELNÉ)

⚠️ **POZNÁMKA:** Tato fáze NENÍ součástí zlatého standardu pro recognition. Slouží pro specifické potřeby programu `recognize_stamp.py` v UC-2 workflow (verify pending scans).

Desátá fáze vytvoří nebo aktualizuje záznam analyzovaného skenu v databázové tabulce `reference_front`. Pokud sken už v databázi existuje (identifikovaný pomocí `file_path`), aktualizují se jeho metadata (SHA256 hash, rozlišení, uploaded_by). Pokud sken neexistuje, vytvoří se nový pending záznam s `confirmed=0`.

**Účel:**
Program `recognize_stamp.py` může být použit jak pro rozpoznávání nových neznámých známek (běžný use case), tak pro analýzu pending skenů před jejich zařazením do Ground Truth databáze (UC-2 workflow). V druhém případě je potřeba vytvořit databázový záznam který expert později potvrdí pomocí `gt_file_manager.py --action verify`.

**Co se ukládá:**
- `file_path`: relativní cesta k původnímu skenu (ne k warpu!)
- `file_sha256`: SHA256 hash PŮVODNÍHO skenu (ne warpu)
- `resolution_w`, `resolution_h`: rozměry PŮVODNÍHO skenu
- `stamp_type_id`: typ známky z auto-detect (nebo NULL pokud manual input)
- `plate_id`, `zp_no`: 0 (pending - bude vyplněno při verify)
- `confirmed`: 0 (pending expert verification)
- `uploaded_by`: username@hostname aktuálního uživatele

**Proč není součástí zlatého standardu:**
GT Management má vlastní upload workflow (UC-1) který správně implementuje file storage s SHA256 naming a proper deduplication. `recognize_stamp.py` vznikl před GT Management a jeho DB UPSERT je legacy feature. Nové implementace by měly použít `gt_upload_utils.py` místo této fáze.

**Technické detaily:**
```python
# Zkontroluj jestli sken už existuje
existing = find_scan(conn, file_path=relative_path)

if existing:
    # UPDATE metadata
    update_scan_metadata(
        conn, existing['id'],
        file_sha256=compute_sha256(input_path),
        resolution_w=orig_bgr.shape[1],
        resolution_h=orig_bgr.shape[0],
        uploaded_by=f"{username}@{hostname}"
    )
else:
    # INSERT nový pending záznam
    scan_id = insert_scan(
        conn,
        file_path=relative_path,
        stamp_type_id=stamp_type_id,
        plate_id=0,  # pending
        zp_no=0,     # pending
        confirmed=0,
        file_sha256=compute_sha256(input_path),
        resolution_w=orig_bgr.shape[1],
        resolution_h=orig_bgr.shape[0],
        uploaded_by=f"{username}@{hostname}"
    )
```

### Fáze 11: Auto-load notes - Načtení poznámek (VOLITELNÉ)

⚠️ **POZNÁMKA:** Tato fáze NENÍ součástí zlatého standardu pro recognition. Slouží pro specifické potřeby programu `recognize_stamp.py` v UC-3 workflow (pending notes management).

Jedenáctá fáze načítá poznámky a metadata z .notes.json souboru pokud existuje vedle analyzovaného skenu. Tyto poznámky (např. "zakoupeno z aukce Burda, lot 123" nebo "poškozený levý okraj") se uloží do databázového záznamu a .notes.json soubor se smaže.

**Účel:**
Při přidávání nových skenů do pending queue pomocí `gt_file_manager.py --action add --note "..."` se poznámky ukládají do .notes.json souboru vedle skenu. Při prvním spuštění `recognize_stamp.py` na takový sken se tyto poznámky přenesou do databáze aby byly dostupné expertovi při verify workflow.

**Formát .notes.json:**
```json
{
  "note": "zakoupeno z aukce Burda 2025-11-28, lot 123",
  "source": "Burda",
  "added_at": "2025-11-28T14:32:00"
}
```

**Co se děje:**
Pokud `{input_path}.notes.json` existuje, načte se JSON obsah, extrahují se klíče `note` a `source` a uloží se do databázového záznamu pomocí `update_scan_metadata(conn, scan_id, notes=note, source=source)`. Po úspěšném uložení se .notes.json soubor smaže aby nedošlo k duplikaci dat.

**Proč není součástí zlatého standardu:**
Toto je specifická feature pro workflow `gt_file_manager.py` + `recognize_stamp.py`. Obecný recognition systém nemá co dělat s notes management - poznámky jsou business logic GT Management subsystému.

### Fáze 12: Output - Výpis TOP-K výsledků

Dvanáctá fáze zobrazuje výsledky rozpoznávání v uživatelsky čitelném formátu. Vypíše se TOP-K kandidátů (typicky K=10) seřazených podle průměrné podobnosti, včetně agregačních statistik pro každého kandidáta. Dále se vypíše SCAN INFO s metadaty analyzovaného skenu pro účely verify workflow.

**Formát výpisu TOP-K kandidátů:**
```
================================================================================
TOP-10 KANDIDÁTI (podle průměrné podobnosti):
================================================================================
#1   500h červená                      TD  1  ZP  42  mean=0.9234  median=0.9189  min=0.8876  max=0.9456  n=5
#2   500h červená                      TD  1  ZP  43  mean=0.9187  median=0.9145  min=0.8823  max=0.9401  n=5
#3   500h červená                      TD  1  ZP  41  mean=0.9156  median=0.9112  min=0.8789  max=0.9378  n=5
...
```

**Výklad metrik:**
- **mean**: průměrná cosine similarity (hlavní metrika pro ranking)
- **median**: medián similarity (robustnější vůči outliers)
- **min/max**: rozsah similarity (ukazuje variabilitu matchů)
- **n**: počet cropů které byly porovnané

**SCAN INFO výpis:**
```
================================================================================
SCAN INFO (pro verify):
================================================================================
scan_id:      6902
file_path:    data/pending/2025-11/scan_001.jpg
file_sha256:  2f9186b896fb...
resolution:   2400×3000
uploaded_by:  milan@zenbook
notes:        zakoupeno z aukce Burda, lot 123
source:       Burda

Pro verify použij:
  python tools/gt_file_manager.py --action verify \
      --scan-id 6902 \
      --plate 1 \
      --zp 42 \
      --env dev
```

**Účel SCAN INFO:**
Když expert kontroluje výsledky recognition a vidí že TOP-1 kandidát je správný, potřebuje `scan_id` pro verify příkaz. SCAN INFO poskytuje všechny potřebné údaje včetně ready-to-copy command line pro verify workflow.

### Fáze 13: Validation - Kontrola očekávaného výsledku (VOLITELNÉ)

Třináctá fáze kontroluje jestli očekávaný výsledek (pokud byl zadán pomocí `--expect-plate` a `--expect-zp` parametrů) je v seznamu TOP-K kandidátů a na jaké pozici. Tato fáze se používá pouze při testování přesnosti systému na známých datech.

**Účel:**
Při batch testování recognition accuracy máme dataset kde známe správnou odpověď (ground truth). Například víme že `scan_500h_TD1_ZP042.jpg` obsahuje známku TD1 ZP42. Spustíme recognition s `--expect-plate 1 --expect-zp 42` a systém vypíše jestli správná odpověď je na pozici #1, #2, ... nebo není v TOP-K vůbec.

**Výstup:**
```
[INFO] Očekávaný kandidát 500h červená TD 1 ZP 42 je na pozici #1
```

nebo

```
[WARN] Očekávaný kandidát 500h červená TD 1 ZP 42 NENÍ v seznamu!
```

**Použití:**
```bash
# Test accuracy na známém skenu
python recognize_stamp.py \
    --env dev \
    --input baseline/500h_TD1_ZP042.jpg \
    --denomination 500h \
    --expect-plate 1 \
    --expect-zp 42

# Batch test celého datasetu
for scan in baseline/*.jpg; do
    # Parse expected TD and ZP from filename
    # Run recognition with --expect-plate and --expect-zp
    # Aggregate results: success if rank=#1, failure otherwise
done
```

### Fáze 14: Visualization - Mozaika porovnání (VOLITELNÉ)

Čtrnáctá fáze vytvoří vizuální mozaiku která vedle sebe zobrazuje analyzovaný sken a TOP-K nejpodobnějších referenčních skenů. Mozaika se uloží jako PNG soubor pro vizuální kontrolu výsledků.

**Účel:**
Čísla similarity scores jsou abstraktní - 0.9234 vs 0.9187 není intuitivní rozdíl. Vizuální mozaika umožňuje expertovi rychle vidět jestli TOP-K kandidáti opravdu vypadají podobně jako analyzovaná známka. To je užitečné při ladění systému nebo při kontrole edge cases.

**Formát mozaiky:**
```
┌─────────────┬─────────────┬─────────────┐
│   QUERY     │   RANK #1   │   RANK #2   │
│  (warped)   │  (warped)   │  (warped)   │
│             │ TD1 ZP42    │ TD1 ZP43    │
│             │ mean=0.9234 │ mean=0.9187 │
├─────────────┼─────────────┼─────────────┤
│  RANK #3    │   RANK #4   │   RANK #5   │
│  (warped)   │  (warped)   │  (warped)   │
│ TD1 ZP41    │ TD1 ZP45    │ TD2 ZP12    │
│ mean=0.9156 │ mean=0.9134 │ mean=0.9089 │
└─────────────┴─────────────┴─────────────┘
```

**Použití:**
```bash
python recognize_stamp.py \
    --env dev \
    --input scan.jpg \
    --denomination 500h \
    --topk 10 \
    --compare-full  # ← Aktivuje vizualizaci
```

**Výstup:**
PNG soubor v `{env}/data/outputs/recognition/compare_{input_stem}_top{K}.png`. Mozaika má fixní velikost panelů (např. 300×250 px per panel) a automaticky se layoutuje do grid.

---

## 🔑 KLÍČOVÉ KONCEPTY

### Proč používáme multiple crops místo celé známky

Rozpoznávání známky jako celku (single embedding pro celou známku) by bylo jednodušší implementačně, ale výrazně méně přesné. Důvod je v tom že různé části známky obsahují různé typy informací které slouží k rozlišení na různých úrovních.

**Hodnotový štítek** obsahuje vytištěnou hodnotu (např. "500") a slouží primárně k určení typu známky (denomination). Je to nejdůležitější crop pro auto-detect workflow protože kombinace embedding + OCR na štítku dosahuje velmi vysoké přesnosti.

**Spirály** (levá a pravá) jsou dekorativní elementy které se liší mezi známkovými poli v rámci stejné tiskové desky. Například na TD1 má ZP42 spirálu otevřenou zatímco ZP43 má spirálu zavřenou. Embedding spirály tedy rozlišuje především pozici na desce.

**Obloha** (pravá horní oblast) má jemnou texturu která je specifická pro každé známkové pole. Tato textura vzniká kombinací tlakové síly při tisku a mikroskopických nerovností na papíru. Je to jeden z nejsilnějších rysů pro rozlišení ZP pozic.

**Keř s větvemi** (levá dolní oblast) rozlišuje především tiskové desky. Různé desky mají mírně odlišné pozice větví nebo jejich sílu, což se projevuje v embedding representaci.

**Číslice a písmena** (pokud jsou na známce) slouží k OCR verifikaci a disambiguation mezi podobnými typy (10h5 vs 15h, 50h15 vs 50h16).

Porovnáváním každého cropu samostatně dostáváme vektor podobností místo single scalar. Například pro kandidáta TD1 ZP42 můžeme mít: hodnotovy_stitek=0.95 (silný match type), spirala_levy=0.89 (střední match pozice), obloha=0.93 (silný match pozice), ker_vetve=0.91 (silný match desky). Agregací těchto podobností dostáváme robustnější klasifikaci než single embedding.

### Embedding vs OCR vs HSV - proč všechny tři metody

Recognition pipeline používá kombinaci tří různých metod proto že každá má své silné a slabé stránky a dohromady se navzájem doplňují.

**Embedding similarity** je velmi robustní vůči variabilitě ve skenu - různé osvětlení, mírné poškození, nebo nečistoty na známce nemají velký vliv na embedding reprezentaci. ResNet50 je natrénovaný na obrovském datasetu a naučil se extrahovat high-level features které jsou invariantní vůči těmto distortions. Embedding similarity dosahuje excelentních výsledků na většině známek, ale má problém s velmi podobnými typy které se liší pouze v číslici (10h5 vs 15h) nebo barvě (50h15 modrá vs 50h16 zelená).

**OCR (Optical Character Recognition)** čte text z hodnotového štítku a převádí ho na string který můžeme parsovat. Pro disambiguation mezi 10h5 a 15h stačí přečíst "10" vs "15". OCR je velmi přesné na kvalitních skenech ale selhává na poškozených nebo špatně vytištěných štítcích. Například 1000h známky často čte jako "100" nebo "10O" protože nuly splývají. OCR alone by tedy nedosahoval dobré accuracy.

**HSV color matching** porovnává průměrnou barvu známky v HSV color space s definovanými pravidly pro každý typ. Pro disambiguation mezi 50h15 (modrá) a 50h16 (zelená) stačí zkontrolovat Hue channel. HSV matching je rychlé a spolehlivé pro barevnou variantu ale nedokáže rozlišit mezi typy se stejnou barvou.

Kombinace všech tří metod (multi-phase detection) dosahuje 96.9% success rate na testovacím datasetu 6800 skenů. Embedding poskytuje kandidáty, OCR disambiguuje číslice, HSV verifikuje barvu. Žádná metoda alone by nedosáhla takové accuracy.

### Deterministické chování - proč je to důležité

Pipeline je navržená tak aby stejný vstup vždy produkoval stejný výstup. To znamená že pokud spustíme recognition na stejném skenu dvakrát (se stejnými parametry), dostaneme naprosto identické výsledky - stejné TOP-K kandidáty ve stejném pořadí se stejnými similarity scores až na poslední desetinné místo.

Deterministické chování je kritické ze tří důvodů:

**Reprodukovatelnost testování:** Když měříme přesnost systému na testovacím datasetu, potřebujeme aby výsledky byly reprodukovatelné. Pokud bychom dnes naměřili 96.9% success rate a zítra 95.2% na stejném datasetu (kvůli random variabilitě), nevěděli bychom jestli jsme systém zlepšili nebo zhoršili. Deterministické chování umožňuje spolehlivé A/B testování změn.

**Debugging a analýza:** Když recognition selže (vrátí špatný výsledek), potřebujeme být schopni přesně reprodukovat problém aby ho šlo analyzovat. S non-deterministickým systémem by stejný sken někdy fungoval správně a někdy ne, což by debugging ztížilo až znemožnilo.

**Konzistence v produkci:** V produkčním nasazení očekáváme že pokud uživatel nahraje stejný sken vícekrát, dostane stejný výsledek. Non-deterministické chování by vedlo k frustraci uživatelů ("včera to říkalo TD1 ZP42, dnes to říká TD1 ZP43").

Deterministické chování zajišťujeme několika mechanismy:

**Fixní seed pro projekční hlavu:** Projekční hlava embedding modelu je inicializovaná s `torch.manual_seed(12345)` před vytvořením vrstev. Tím zajistíme že všechny random inicializace váh jsou identické.

**L2 normalizace embeddingů:** Všechny embedding vektory jsou L2-normalizované na normu 1.0. To eliminuje variabilitu v délce vektoru a zajišťuje že cosine similarity je deterministická operace.

**Deterministický warp:** Homografická matice pro warp transformaci je vypočítaná z quad pomocí fixed point arithmetic, takže numerické zaokrouhlovací chyby jsou minimální a konzistentní.

**Žádné random augmentace:** Na rozdíl od trénování, v inference módu nepoužíváme žádné random augmentace (flip, rotate, color jitter). Vstup je zpracován vždy stejně.

### L2 normalizace a cosine similarity

Všechny embedding vektory (jak query tak reference) jsou L2-normalizované což znamená že jejich euklidovská norma je přesně 1.0. Matematicky: `||v|| = sqrt(v[0]^2 + v[1]^2 + ... + v[511]^2) = 1.0`.

L2 normalizace má dva důležité důsledky:

**Cosine similarity = dot product:** Pro dva L2-normalizované vektory a, b platí: `cosine_similarity(a, b) = dot(a, b) / (||a|| * ||b||) = dot(a, b) / (1.0 * 1.0) = dot(a, b)`. Cosine similarity se tedy redukuje na prostý skalární součin (dot product). Dot product je výrazně rychlejší než výpočet cosine similarity - na CPU jde o řádově 10× speedup, na GPU ještě více.

**Hodnoty v rozsahu [-1, 1]:** Dot product dvou L2-normalizovaných vektorů je vždy v rozsahu [-1, 1] kde 1 znamená identické vektory (úhel 0°), 0 znamená ortogonální vektory (úhel 90°), -1 znamená opačné vektory (úhel 180°). V praxi se embedding vektory známek pohybují v rozsahu 0.7-0.98 - nikdy nejsou opačné nebo ortogonální.

L2 normalizace se provádí po projekci z 2048D na 512D:
```python
embedding = projection_head(resnet_features)  # (512,)
embedding = embedding / torch.norm(embedding)  # L2 normalize
```

Všechny referenční embeddings v databázi jsou uložené už L2-normalizované takže při matchingu stačí načíst vektor z BLOB a spočítat dot product - není potřeba žádná další normalizace.

### Agregační statistiky - mean, median, min, max

Pro každého kandidáta (kombinaci TD + ZP) máme několik similarity scores - jeden pro každý crop který byl matchovaný. Tyto scores agregujeme do statistik které popisují celkovou podobnost kandidáta.

**Mean (průměr)** je primární metrika pro ranking kandidátů. Vypočítá se jako aritmetický průměr všech similarity scores pro daného kandidáta. Mean dává všem crops stejnou váhu - spirála, obloha a keř jsou stejně důležité. Kandidáti jsou seřazení sestupně podle mean - čím vyšší průměr, tím podobnější kandidát.

**Median (medián)** je robustnější než mean vůči outliers. Pokud má kandidát jeden crop s velmi nízkou similarity (outlier) zatímco ostatní jsou vysoké, mean klesne výrazně ale median zůstane vysoký. Median je užitečný pro identifikaci kandidátů kde většina crops matchuje dobře ale jeden či dva jsou off.

**Min (minimum)** ukazuje nejhorší match mezi všemi crops. Nízké minimum (například < 0.80) naznačuje že alespoň jeden crop matchuje špatně, což může být red flag pro false positive. Naopak vysoké minimum (> 0.88) naznačuje konzistentně dobré matchování across all crops.

**Max (maximum)** ukazuje nejlepší match. Vysoké maximum (> 0.95) znamená že alespoň jeden crop matchuje velmi dobře. Ale pozor - high max s low min naznačuje nekonzistentní matching který může být false positive.

**Std (směrodatná odchylka)** měří variabilitu similarity scores. Nízká std (< 0.02) znamená že všechny crops mají podobnou similarity - kandidát matchuje konzistentně. Vysoká std (> 0.05) znamená velké rozdíly mezi crops - některé matchují dobře, jiné špatně. Vysoká std je warning sign.

**n (count)** je počet crops které byly porovnané. Typicky n=4-8 podle drawing_id. Pokud je n výrazně nižší než expected (například n=2 místo 5), znamená to že některé crops chyběly v databázi nebo matchování selhalo.

Příklad interpretace:
```
Kandidát A: mean=0.92, median=0.91, min=0.89, max=0.94, std=0.018, n=5
→ Velmi dobrý match - všechny crops konzistentně vysoké

Kandidát B: mean=0.88, median=0.89, min=0.75, max=0.95, std=0.078, n=5
→ Nekonzistentní match - některé crops dobře, jiné špatně
→ Možný false positive nebo damaged scan
```

---

## ⚠️ DŮLEŽITÁ PRAVIDLA

### Pravidlo 1: Preprocessing MUSÍ být identický s compute_reference_embeddings.py

Recognition pipeline a program `compute_reference_embeddings.py` (který vypočítává referenční embeddings pro databázi) musí používat naprosto identické preprocessing kroky. Jakýkoliv rozdíl v preprocessing vede k neporovnatelným embeddingům a recognition selže.

**Proč je to kritické:**
Embedding reprezentace závisí na tom jak byl vstupní crop preprocessovaný. Pokud referenční embeddings byly vypočítané s jedním preprocessing (např. normalizace mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]) a query embeddings s jiným (např. normalizace [0, 1]), vektory leží v různých regionech 512D prostoru a jejich podobnost není interpretovatelná.

**Co musí být identické:**
- **Warp target size:** 1300×1100 pixelů (pevně dané)
- **Color space:** BGR (OpenCV default), konverze BGR→RGB uvnitř encoderu
- **Crop extraction:** Stejné relativní souřadnice, padding, polarity
- **Resize:** 224×224 před ResNet50 (pevně dané ResNet50 vstupem)
- **Normalizace:** Pouze dělení 255.0 → rozsah [0, 1], ŽÁDNÁ ImageNet normalizace
- **Embedding model:** Stejný ResNet50 model se stejnou projekční hlavou (stejný seed!)
- **L2 normalizace:** Aplikovaná vždy na konci

**Jak zajistit konzistenci:**
Oba programy používají stejný `RealEncoder` class z `common/embedding_utils.py` který má preprocessing zabudovaný uvnitř. To zajišťuje že preprocessing je vždy identický - není možné náhodou použít jiné parametry.

**Kontrola konzistence:**
Při změně preprocessing (například změna warp size nebo normalizace) je NUTNÉ přepočítat všechny referenční embeddings v databázi. Nelze míchat staré embeddings s novými - buď vše staré nebo vše nové.

**✅ Správně:**
```python
# recognize_stamp.py
from common.embedding_utils import RealEncoder
encoder = RealEncoder(model_path, head_path, seed=12345)
query_emb = encoder(crop_img)  # Preprocessing uvnitř

# compute_reference_embeddings.py
from common.embedding_utils import RealEncoder
encoder = RealEncoder(model_path, head_path, seed=12345)
ref_emb = encoder(crop_img)  # STEJNÝ preprocessing
```

**❌ Špatně:**
```python
# recognize_stamp.py - custom preprocessing
crop_normalized = (crop_img / 255.0 - [0.485, 0.456, 0.406]) / [0.229, 0.224, 0.225]
query_emb = model(crop_normalized)

# compute_reference_embeddings.py - jiný preprocessing
crop_normalized = crop_img / 255.0  # Jiná normalizace!
ref_emb = model(crop_normalized)

# → Embeddings neporovnatelné!
```

### Pravidlo 2: Projekční hlava MUSÍ být načtena z .pth souboru

Projekční hlava (2048D → 512D neuronová síť) musí být VŽDY načtena z .pth souboru který obsahuje předtrénované váhy. Nikdy nesmí být inicializována náhodně nebo jinak než z .pth souboru uloženého v databázi.

**Proč je to kritické:**
Projekční hlava je natrénovaná neuronová síť která se naučila optimální redukci dimenze pro rozlišování československých známek. Náhodná inicializace by dala kompletně jiné embeddings které nemají žádný vztah k embeddings v databázi. Recognition by pak fungoval náhodně - jako házení mincí.

**Jak funguje načítání:**
```python
# 1. Načti head_path a seed z databáze (model_embed_head tabulka)
head_row = conn.execute(
    "SELECT head_path, seed FROM model_embed_head WHERE model_id = ?",
    (model_id,)
).fetchone()

# 2. Nastav fixní seed pro deterministickou strukturu
torch.manual_seed(head_row['seed'])  # Typicky 12345

# 3. Vytvoř strukturu sítě (vrstvy)
head = nn.Sequential(
    nn.Linear(2048, 512),
    nn.ReLU(),
    nn.Dropout(0.1)
)

# 4. Načti trénované váhy z .pth souboru
head.load_state_dict(torch.load(head_row['head_path']))

# 5. Nastav eval() mód
head.eval()
```

**Seed je důležitý pro strukturu:**
Seed se nastavuje PŘED vytvořením vrstev protože některé vrstvy (například Dropout) používají random inicializaci. Se stejným seedem dostaneme stejnou strukturu. Ale pozor - seed ovlivní jen strukturu, ne váhy! Váhy jsou načtené z .pth souboru a ty už seed neovlivní.

**Verifikace správnosti:**
.pth soubor má v databázi uložený SHA256 hash. Před použitím můžeme verifikovat že soubor je nepoškozený:
```python
import hashlib

with open(head_path, 'rb') as f:
    file_hash = hashlib.sha256(f.read()).hexdigest()

if file_hash != head_row['head_sha256']:
    raise ValueError("Projection head file corrupted or wrong version!")
```

**✅ Správně:**
```python
torch.manual_seed(12345)
head = create_projection_head()
head.load_state_dict(torch.load('head.pth'))
```

**❌ Špatně:**
```python
torch.manual_seed(12345)
head = create_projection_head()
# Chybí load_state_dict! Používá random váhy!
```

### Pravidlo 3: ŽÁDNÁ ImageNet normalizace v inference

Při výpočtu embeddingů v inference módu NIKDY nesmíme použít ImageNet normalizaci (mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]). Používáme pouze prostou normalizaci dělením 255.0 která převede pixely z rozsahu [0, 255] do rozsahu [0, 1].

**Proč je to kritické:**
ImageNet normalizace je standardní preprocessing pro modely trénované na ImageNet datasetu. Ale náš embedding model byl fine-tunovaný na československé známky s jinou normalizací. Pokud bychom použili ImageNet normalizaci, preprocessing by se lišil od toho který byl použit při fine-tuningu a embeddings by byly špatné.

**Co je ImageNet normalizace:**
```python
# ImageNet normalizace (NEPOUŽÍVAT!)
mean = [0.485, 0.456, 0.406]  # RGB channels
std = [0.229, 0.224, 0.225]
normalized = (image - mean) / std
```

Tato normalizace posunuje a škáluje pixely aby měly podobné statistiky jako ImageNet dataset. Ale naše známky mají jiné statistiky - například více šedých tónů a méně saturovaných barev než fotky z ImageNetu.

**Co místo toho používáme:**
```python
# Naše normalizace (SPRÁVNĚ)
normalized = image / 255.0  # [0, 255] → [0, 1]
```

Prostá normalizace do rozsahu [0, 1] zachovává relativní intenzity pixelů beze změny. To je přesně to co chceme - embeddings jsou pak konzistentní s referenčními embeddings které byly vypočítané stejným způsobem.

**Kde je normalizace implementovaná:**
Normalizace je zabudovaná uvnitř `RealEncoder.__call__()` metody takže programátor ji nemůže náhodou udělat špatně:
```python
class RealEncoder:
    def __call__(self, crop_bgr: np.ndarray) -> np.ndarray:
        # BGR → RGB
        crop_rgb = cv2.cvtColor(crop_bgr, cv2.COLOR_BGR2RGB)
        
        # Resize
        crop_resized = cv2.resize(crop_rgb, (224, 224))
        
        # Normalizace [0,255] → [0,1]
        crop_float = crop_resized.astype(np.float32) / 255.0  # ← TADY
        
        # PyTorch tensor
        crop_tensor = torch.from_numpy(crop_float).permute(2, 0, 1)
        
        # Forward pass
        ...
```

**✅ Správně:**
```python
crop_normalized = crop_bgr / 255.0
```

**❌ Špatně:**
```python
mean = [0.485, 0.456, 0.406]
std = [0.229, 0.224, 0.225]
crop_normalized = (crop_bgr / 255.0 - mean) / std  # NEPOUŽÍVAT!
```

### Pravidlo 4: Matching pouze v rámci stejné kresby (drawing_id)

Při matching query embeddingů s referenčními embeddingy MUSÍME porovnávat pouze embeddings ze stejné kresby (drawing_id). Nelze porovnávat známku s kresbou "s popisem" proti známce s kresbou "abstraktní".

**Proč je to kritické:**
Různé kresby mají různé crops - například kresba "s popisem" má crop na pozici nápisu POŠTA, zatímco kresba "abstraktní" tento crop nemá. Embeddings různých crops leží v různých regionech 512D prostoru a jejich podobnost není interpretovatelná. Porovnávat "nápis POŠTA" vs "spirála" by bylo jako porovnávat jablka s pomeranči.

**Jak se určuje drawing_id:**
Drawing_id je odvozené z denomination (typu známky). Každý typ známky má přiřazenou konkrétní kresbu v databázové tabulce `stamp_types`:
```sql
SELECT drawing_id FROM stamp_types WHERE id = stamp_type_id;
```

Například známka 500h má drawing_id=5 (abstraktní kresba), známka 10h5 má drawing_id=1 (s popisem). Po určení denomination (buď manuálně nebo auto-detect) víme drawing_id a můžeme načíst příslušné crops.

**Enforcement v SQL query:**
```sql
-- Matching query embedding crop_id=8 (spirala_levy)
SELECT 
    re.scan_id,
    rf.plate_id,
    rf.zp_no,
    dot_product(re.vec, ?) AS similarity
FROM reference_embeddings re
JOIN reference_front rf ON re.scan_id = rf.id
WHERE 
    re.crop_id = 8                    -- STEJNÝ crop
    AND rf.drawing_id = ?             -- ← STEJNÁ kresba (enforcement)
    AND rf.confirmed = 1
ORDER BY similarity DESC;
```

Drawing_id je předaný jako parametr do query aby se filtroval pouze embeddings ze stejné kresby.

**Co se stane při porušení:**
Pokud bychom matchovali napříč kresbami, dostali bychom nesmyslné výsledky - například známka 500h (abstraktní) by mohla matchovat s 10h5 (s popisem) protože náhodou některé embeddings jsou v podobném regionu prostoru. Recognition by selhal s nepředvídatelnými výsledky.

**✅ Správně:**
```python
# Query má drawing_id=5 (500h, abstraktní)
# Matching pouze proti drawing_id=5
matches = match_embeddings(conn, query_embs, drawing_id=5)
```

**❌ Špatně:**
```python
# Matching proti VŠEM drawing_id - CHYBA!
matches = match_embeddings(conn, query_embs, drawing_id=None)
```

---

## 🐛 ZNÁMÉ PROBLÉMY / OMEZENÍ

### Problém 1: OCR selhává na 1000h známkách (58% přesnost)

OCR model (EasyOCR) má výrazně nižší přesnost na známkách 1000h ve srovnání s ostatními typy. Na testovacím datasetu dosahuje pouze 58% úspěšnosti rozpoznání textu "1000" z hodnotového štítku. U ostatních denominací je OCR přesnost typicky > 95%.

**Proč to selhává:**
Hodnotový štítek na 1000h známce obsahuje čtyři číslice "1000" vytištěné relativně malým fontem. Nuly často splývají dohromady nebo se čtou jako písmeno "O". EasyOCR model čte text jako "100", "10O0", "1OOO" nebo "IO00" - tedy s různými variacemi záměny nula/O. Pro parsing je pak těžké určit jestli jde o 1000h nebo o něco jiného.

**Dopad na recognition:**
Multi-phase detection používá OCR jako sekundární metodu pro disambiguation. Když OCR selže, spoléháme pouze na embedding similarity. Pro 1000h to znamená že auto-detect může mít nižší confidence nebo v edge cases vybrat špatného kandidáta. V praxi díky vysoké embedding similarity (> 0.90) recognition stále funguje relativně dobře (celková success rate 96.9%), ale není to ideální.

**Možná řešení:**
1. **Větší crop:** Zvětšit crop hodnotového štítku aby číslice byly větší a lépe čitelné
2. **OCR preprocessing:** Předzpracovat crop (contrast enhancement, denoising) před OCR
3. **Specialized OCR:** Natrénovat vlastní OCR model specificky na československé známky
4. **Pattern matching:** Místo obecného OCR použít template matching pro specifické fonty
5. **Trust embeddings:** Při detection 1000h trusted více embedding similarity než OCR

**Workaround:**
Aktuálně používáme workaround kde pokud OCR přečte "100" nebo podobné varianty a embedding similarity je vysoká (> 0.88), přidáváme 1000h jako dodatečného kandidáta. To zlepšilo accuracy z 58% na 78%, ale stále není perfektní.

### Problém 2: 15h TD4 má nízkou úspěšnost (~5%)

Známka 15h tisková deska TD4 má extrémně nízkou recognition accuracy (~5%) ve srovnání s ostatními deskami téže známky (TD1-3 dosahují 95-98%). Jedná se o systematický problém specifický pouze pro TD4.

**Proč to selhává:**
Referenční skeny TD4 v databázi mají výrazně horší kvalitu než TD1-3. Většina TD4 skenů pochází z jednoho zdroje (pravděpodobně amatérský scan) a jsou rozmazané, mají špatné osvětlení nebo nízké rozlišení. Embeddings vypočítané z takových skenů nejsou reprezentativní pro kvalitní TD4 exempláře.

Když pak analyzujeme kvalitní TD4 sken, jeho embeddings se neporovnávají dobře s low-quality referenčními embeddings v databázi. Recognition systém pak matchuje proti TD1-3 které mají quality reference skeny, místo správné TD4.

**Dopad:**
TD4 známky jsou velmi vzácné ve srovnání s TD1-3, takže tento problém ovlivňuje pouze malé procento celkového datasetu. Na celkové accuracy 96.9% má TD4 problém minimální vliv. Ale pro uživatele kteří mají TD4 exemplář je to frustrující - systém jim vrátí špatný výsledek.

**Možná řešení:**
1. **Rescan TD4:** Získat high-quality skeny TD4 exemplářů a nahradit jimi low-quality reference
2. **Quality filtering:** Automaticky detekovat low-quality skeny a vyřadit je z databáze
3. **Quality normalization:** Preprocessing který normalizuje rozdíly v kvalitě (deblurring, enhancement)
4. **Separate handling:** Specifické parametry nebo threshold pro TD4 matching

**Status:**
Priorita LOW protože ovlivňuje pouze malé procento případů. Plánované řešení je rescan TD4 - získat 10-20 high-quality TD4 exemplářů ze spolehlivého zdroje a nahradit jimi současné reference. To by mělo accuracy zlepšit z 5% na očekávaných 95%+.

### Problém 3: False positives při poškozených skenech

Recognition pipeline má tendenci produkovat false positives když je vstupní sken výrazně poškozený, rozmazaný nebo má velké nečistoty. Například sken s velkým záhybem přes střed známky může matchovat s vysokou confidence proti nesprávnému kandidátovi.

**Proč to selhává:**
Embeddings jsou relativně robustní vůči malým defektům (drobné škrábance, dust particles) ale výrazné poškození může změnit embedding reprezentaci natolik že připomíná jinou známku. Pokud je například spirála zakrytá záhybem, její embedding se změní a může náhodou matchovat spirálu z jiného ZP.

Agregační statistiky (mean, median, min, max) částečně pomáhají identifikovat takové případy - pokud má kandidát vysokou varianci (std > 0.05) nebo nízké minimum (< 0.80), je to warning sign. Ale není to dokonalé řešení.

**Dopad:**
Odhadovaná frekvence je < 1% (na 6800 testovacích skenů < 50 false positives). Většinou jde o edge cases kde sken je skutečně v very poor condition. V produkčním nasazení je důležité aby uživatel mohl false positive reportovat pro review.

**Možná řešení:**
1. **Quality detection:** Před recognition detekovat quality skenu a varovat uživatele
2. **Confidence thresholds:** Při nízké confidence (mean < 0.85 or std > 0.05) vrátit "uncertain" místo top candidate
3. **Manual review flag:** Automaticky flagovat suspicious results pro expert review
4. **Multiple scans:** Požádat uživatele o multiple skeny ze stejné známky a agregovat výsledky

**Workaround:**
Aktuálně vizuální mozaika (FÁZE 14) umožňuje uživateli vizuálně zkontrolovat jestli TOP-1 kandidát opravdu vypadá podobně. To není automatické ale alespoň dává možnost odhalit false positive before commit.

---

## 🔗 SOUVISLOSTI

### Implementováno v

**recognize_stamp.py v3.2.0** - Produkční implementace recognition pipeline. Hlavní program pro rozpoznávání známek z command line. Implementuje všech 14 fází pipeline přesně jak je popsáno v tomto dokumentu.

**compute_reference_embeddings.py** - Program pro výpočet referenčních embeddingů které se ukládají do databáze. Používá IDENTICKÉ preprocessing jako recognize_stamp.py aby embeddings byly porovnatelné. Implementuje fáze 1-2, 4, 6-7 (načtení konfigurace, encoder init, warp, crops, embedding computation).

### Používá

**common/embedding_utils.py v1.3.0** - RealEncoder class která zapouzdřuje preprocessing a embedding computation. Obsahuje deterministickou projekční hlavu, BGR→RGB konverzi, normalizaci [0,1] a L2 normalizaci výstupu.

**common/denomination_utils.py v2.0.1** - Multi-phase auto-detect algoritmus který kombinuje embedding similarity, OCR a HSV color matching. Používaný ve FÁZI 5 pro automatickou detekci typu známky.

**common/ocr_utils.py v1.0.0** - EasyOCR wrapper pro extrakci textu z hodnotového štítku. Používaný v rámci auto-detect workflow pro disambiguation podobných typů (10h5 vs 15h).

**common/color_utils.py v1.0.0** - HSV color matching utilities pro verifikaci barvy známky. Používaný jako terciární metoda v auto-detect workflow.

**common/yolo_utils.py v1.0.0** - YOLO detection wrapper pro detekci rámečků známek. Používaný ve FÁZI 3 pro hybrid detection workflow.

**common/db_utils.py v1.1.0** - Centralizované databázové operace (22 funkcí) pro všechny interakce s databází. Eliminuje duplicitní DB kód a zajišťuje konzistentní přístup k datům.

**common/load_config.py v1.9.1** - Path management a konfigurace environment (dev/prod/sandbox). Single source of truth pro všechny cesty k modelům, databázi a datům.

### Navazující dokumenty

**base/database_schema.md** - Kompletní dokumentace databázové struktury včetně tabulek `reference_front`, `reference_embeddings`, `model_registry`, `model_embed_head`, `drawing_crops`. Popisuje jaká data jsou kde uložená a proč.

**decisions/realencoder_centralization.md** - Rozhodnutí proč a jak jsme centralizovali embedding computation do RealEncoder class. Vysvětluje rozdíl mezi RealEncoder (wrapper s preprocessing) a ResNet50Embed (raw model).

**decisions/schema_v3_1_0_nullable_columns.md** - Rozhodnutí o nullable metadata columns v DB schema v3.1.0. Vysvětluje pending workflow kde skeny mají NULL metadata dokud nejsou potvrzeny expertem.

**decisions/gt_management_use_cases.md** - Celkový koncept GT Management systému (17 use cases, 4 workflows). Recognition pipeline je použitý v UC-2 (Confirm Classification) jako nástroj pro analýzu pending skenů.

**decisions/uc1_workflow.md** - Detailní popis UC-1 Upload & Analyze workflow který je komplementární k recognition pipeline. UC-1 řeší file storage a deduplication, recognition pipeline řeší samotné rozpoznávání.

---

**Tento dokument je zlatý standard pro recognition workflow. Všechny nové implementace MUSÍ dodržet fáze 1-9. Fáze 10-14 jsou volitelné a mohou být vynechány nebo nahrazeny podle potřeby use case.**

---

**Poslední aktualizace:** 2026-01-13  
**Autor:** Milan + Claude  
**Zdroj:** recognize_stamp.py v3.2.0  
**Status:** ✅ ACTIVE - PRODUCTION REFERENCE

