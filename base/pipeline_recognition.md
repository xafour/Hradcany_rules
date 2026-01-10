# Recognition Pipeline - Zlatý Standard

**Tags:** `#pipeline` `#recognition` `#zlatý-standard` `#file-handling`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active - Production Reference

---

## 🎯 ÚČEL

Recognition pipeline je hlavní součást systému Hradčany, která automaticky identifikuje poštovní známky ze skenů nebo fotografií. Když uživatel nahraje obrázek známky, pipeline ji zpracuje a vrátí seznam nejpodobnějších kandidátů z Ground Truth databáze. Celý proces je navržen jako deterministický - stejný vstup vždy produkuje stejný výstup, což je klíčové pro spolehlivost a reprodukovatelnost výsledků.

Pipeline řeší problém, že vstupní skeny můžou být různé kvality, různě natočené nebo vyfocené z úhlu. Musíme je standardizovat do jednotného formátu, ze kterého pak můžeme spolehlivě extrahovat charakteristické rysy pro porovnání s referenčními známkami v databázi.

Tento dokument popisuje **zlatý standard** - jak recognition pipeline SPRÁVNĚ funguje v produkční implementaci `recognize_stamp.py` v3.1.1. Je to referenční dokumentace kterou MUSÍ dodržet všechny nové implementace včetně GT Management.

---

## 🔄 HLAVNÍ WORKFLOW

Recognition pipeline zpracovává vstupní sken v devíti na sebe navazujících fázích. Každá fáze má jasně definovaný vstup a výstup a provádí jednu konkrétní úlohu. Tento lineární design jsme zvolili proto, že umožňuje snadné ladění (můžeme zastavit pipeline po libovolné fázi a zkontrolovat mezivýsledek) a také testování každé fáze nezávisle.

### Fáze 1: Načtení konfigurace modelů

Pipeline nejdřív načte z databáze základní konfiguraci: 
která verze YOLO modelu se má použít pro detekci rámečků 
a která verze embedding modelu (včetně projekční hlavy) 
se má použít pro výpočet embeddingů.

---

### Fáze 2: Preprocessing - File Handling 🔴 KRITICKÁ FÁZE

Tato fáze je KRITICKY DŮLEŽITÁ protože definuje jak správně pracovat se soubory. Je to zlatý standard který musí dodržet všechny implementace.

**Správný postup:**

```python
# 1. NAČTI CELÝ ORIGINÁLNÍ SKEN
orig_bgr = cv2.imread(str(in_path))

if orig_bgr is None:
    raise FileNotFoundError(f"Nelze načíst obrázek: {in_path}")

# 2. WARP PRO ZPRACOVÁNÍ (ne pro uložení!)
full_norm = warp_scan(in_path, quad, target=(1300,1100))
```

**CO SE DĚJE:**
- `orig_bgr` je CELÝ originální sken jak ho poskytl uživatel - v plném rozlišení, nemodifikovaný
- `full_norm` je WARP na standardizovaný formát 1300×1100 pixelů - používá se POUZE pro zpracování (výpočet embeddingů)
- Warp je DOČASNÝ - vytvoříme ho, použijeme pro embeddings, a pak ho zahozujeme
- Do databáze a storage ukládáme VŽDY metadata a cestu k ORIGINÁLU, ne k warpu

**PROČ JE TO DŮLEŽITÉ:**
Warp můžeme kdykoli znovu vytvořit (máme uložený quad v `inference_frames`), ale originál po přepsání už nelze obnovit. Ukládání originálu nám dává flexibilitu - v budoucnu můžeme použít jiný warp size nebo lepší algoritmus warpování a re-analyzovat všechny skeny z originálů. Pokud bychom uložili warp, ztratili bychom tuto možnost.

**❌ ŠPATNÝ POSTUP (BUG):**
```python
# TAKHLE NE!
warped_bgr = warp_scan(in_path, quad, target=(1300,1100))
file_sha256 = compute_sha256_from_array(warped_bgr)  # ❌ Hash z warpu!
cv2.imwrite(storage_path, warped_bgr)                # ❌ Uložit warp!
```

Toto je chyba kterou má současná implementace UC-1 v `gt_upload_utils.py`. Viz `tasks/bug_uc1_file_storage.md` pro detaily.

---

### Fáze 3: Auto-detect Denomination (volitelné)

Pokud uživatel nezadal denomination pomocí `--denomination`, pipeline automaticky detekuje o jakou známku jde. Používá multi-phase detection: nejdřív porovná embeddings z hodnotového štítku s embeddingy všech 26 denominations, pak použije OCR pro disambiguaci mezi podobnými čísly, a nakonec zkontroluje barvu pomocí HSV pravidel.

Auto-detection vrací confidence level (HIGH/MEDIUM/LOW) a denomination (např. "500h"). Pokud je confidence LOW, pipeline se zastaví a požaduje manuální zadání denomination - nechceme pokračovat s nejistou klasifikací.

---

### Fáze 4: Extrakce výřezů

Pro danou `drawing_id` (určenou z denomination) načte pipeline definice required crops z tabulky `drawing_crops`. Crops jsou oblasti na známce které obsahují charakteristické rysy - například spirála vlevo, obloha, keř s větvemi, hodnotový štítek. Z warpnuté známky (`full_norm`) se tyto oblasti vyříznou podle definovaných souřadnic.

Proč používáme multiple crops místo celé známky? Protože různé části známky obsahují různé typy informací. Spirála rozlišuje tiskové desky, hodnotový štítek obsahuje čísla (denomination), obloha má texturu která je specifická pro danou kresbu. Porovnáváním každého cropu samostatně dostáváme jemnější rozlišení.

---

### Fáze 5: Výpočet embeddingů

Pro každý vyříznutý crop pipeline vypočítá 512-dimenzionální embedding vector pomocí RealEncoder. RealEncoder je wrapper kolem ResNet50 který má zabudované preprocessing (BGR→RGB, resize na 224×224, normalizace) a deterministickou projekční hlavu (2048D→512D, seed=12345). Output je L2-normalizovaný vektor - má normu 1.0, což umožňuje rychlé porovnání pomocí dot product.

Je kriticky důležité že používáme STEJNÝ encoder (RealEncoder s STEJNÝM seedem) jaký byl použit pro výpočet referenčních embeddingů v databázi. Jakýkoliv rozdíl v preprocessing nebo projekční hlavě by vedl k neporovnatelným embeddingům.

---

### Fáze 6: Matching - Cosine Similarity

Pro každý query embedding pipeline najde TOP-K nejpodobnějších referenčních embeddingů v databázi. Protože embeddings jsou L2-normalizované, cosine similarity = dot product, což je velmi rychlá operace.

Pipeline porovnává embeddings v rámci STEJNÉ `drawing_id` - známka s kresbou "s popisem" se neporovnává s známkami s kresbou "abstraktní". Důvod: různé kresby mají jiné crops, embeddings by nebyly srovnatelné.

---

### Fáze 7: Agregace kandidátů

Pro každý referenční sken pipeline má několik similarity scores (jeden pro každý crop). Tyto scores se agregují - vypočítá se mean, median, min, max similarity. Výsledkem je jeden aggregate score pro každou kombinaci (tisková deska, známková pozice).

Kandidáti se seřadí podle aggregate score (typicky mean similarity) sestupně. TOP-1 kandidát je nejpravděpodobnější match.

---

### Fáze 8: DB UPSERT (volitelné) 🔴 KRITICKÁ FÁZE

Pokud byl sken nově načtený (ne z databáze), pipeline může vytvořit nebo updatovat záznam v `reference_front`. Toto je druhá kritická fáze pro file handling.

**Správný postup:**

```python
# 1. COMPUTE SHA256 Z ORIGINÁLU
file_sha256 = compute_sha256(in_path)  # in_path je cesta k ORIGINÁLU

# 2. ZJISTI RESOLUTION Z ORIGINÁLU  
resolution_h, resolution_w = orig_bgr.shape[:2]

# 3. FIND OR INSERT
existing = find_scan(conn, file_path=rel_path_str)

if existing:
    # UPDATE existujícího záznamu
    scan_id = existing['id']
    update_scan_metadata(
        conn, scan_id,
        file_sha256=file_sha256,      # ← Hash originálu
        resolution_w=resolution_w,    # ← Rozměry originálu
        resolution_h=resolution_h,
        uploaded_by=get_current_user()
    )
else:
    # INSERT nového záznamu
    scan_id = insert_scan(
        conn,
        file_path=Path(rel_path_str),   # ← Cesta k ORIGINÁLU
        file_sha256=file_sha256,        # ← Hash ORIGINÁLU
        resolution_w=resolution_w,      # ← Rozměry ORIGINÁLU
        resolution_h=resolution_h,
        uploaded_by=get_current_user()
    )
```

**KLÍČOVÉ BODY:**
- SHA256 se počítá z ORIGINÁLU, ne z warpu
- Resolution se čte z `orig_bgr` (originál), ne z `full_norm` (warp)
- `file_path` ukazuje na ORIGINÁL

---

### Fáze 9: Output

Pipeline vrátí TOP-K kandidátů seřazených podle aggregate similarity. Každý kandidát obsahuje: denomination, tiskovou desku, známkovou pozici, similarity score, a další metadata. Output je formátovaný do čitelné tabulky nebo JSON struktury podle parametru `--output`.

---

## 🔑 KLÍČOVÉ MANTRY

### 1. VŽDY ulož ORIGINÁL, ne warp

```
✅ Storage: original.jpg (celý sken jak ho user nahrál)
❌ Storage: warp_1300x1100.jpg (zpracovaný)
```

Warp je DOČASNÝ artefakt zpracování. Vytvoříme ho, použijeme pro embeddings, zahozujeme. Storage obsahuje VŽDY originály.

### 2. Warp JEN pro zpracování

```python
orig_bgr = cv2.imread(path)       # Originál - uložíme
full_norm = warp_scan(...)        # Warp - použijeme a zahodíme
embeddings = encoder(crops)       # Vypočítáme z warpu
# full_norm už dál nepotřebujeme
```

### 3. SHA256 VŽDY z originálu

```python
file_sha256 = compute_sha256(original_path)  # ✅ Správně
file_sha256 = compute_sha256(warp_array)     # ❌ Chyba!
```

Důvod: deduplication. Dva uživatelé kteří nahrají stejný originál musí dostat stejný hash.

### 4. DB ukládá metadata ORIGINÁLU

```python
insert_scan(
    file_path=original_path,        # ✅ Cesta k originálu
    file_sha256=hash(original),     # ✅ Hash originálu
    resolution_w=original.width,    # ✅ Rozměry originálu
)
```

---

## ⚠️ ZNÁMÝ BUG

**UC-1 File Storage Bug** v `gt_upload_utils.py` v1.6.0:
- Ukládá warp místo originálu
- Počítá SHA256 z warpu
- Ukládá rozměry warpu (1300×1100) místo originálu

**Fix:** Použít workflow z této dokumentace (recognize_stamp.py)

**Detaily:** [tasks/bug_uc1_file_storage.md](../tasks/bug_uc1_file_storage.md)

---

## 📊 PERFORMANCE

- **96.9% success rate** na baseline datasetu (6800 skenů)
- **100% accuracy** pro 8 denominations
- **99%+ accuracy** pro 10 denominations
- **Known issues:** 1000h OCR (58%), 15h TD4 (87%)

---

## 🔗 SOUVISLOSTI

**Implementováno v:**
- `recognize_stamp.py` v3.1.1 - Production implementation

**Používá:**
- `common/embedding_utils.py` - RealEncoder
- `common/denomination_utils.py` - Auto-detect
- `common/db_utils.py` - Database operations

**Souvisí s:**
- [base/database_schema.md](./database_schema.md) - reference_front tabulka
- [decisions/uc1_file_storage_bug.md](../decisions/uc1_file_storage_bug.md) - Známý bug ⭐
- [decisions/uc1_workflow.md](../decisions/uc1_workflow.md) - UC-1 musí dodržet tento workflow

---

**VŠECHNY nové implementace (GT Management, API, etc.) MUSÍ dodržet tento workflow!**

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Zdroj:** recognize_stamp.py v3.1.1  
**Status:** ✅ PRODUCTION REFERENCE - DO NOT DEVIATE

---
