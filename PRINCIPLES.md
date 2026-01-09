# 💻 KÓDOVACÍ PRINCIPY

**Verze:** 0.9  
**Datum:** 2026-01-06  

---

## 🎯 ÚČEL

Tento dokument definuje kódovací principy a best practices pro projekt Hradčany. Na rozdíl od CONSTITUTION.md, který obsahuje neměnná pravidla workflow, tento dokument popisuje konkrétní principy pro psaní kódu.

---

## 📜 ZÁKLADNÍ PRINCIPY

### **PRINCIP #1: Single Source of Truth**

Každá funkce, každý algoritmus, každý kus logiky je implementován pouze jednou v celém projektu.

**Důvod:** Eliminace duplicitního kódu zabraňuje nekonzistenci, usnadňuje údržbu a snižuje riziko chyb.

**Příklad správně:**
```python
# V common/db_utils.py
def get_scan_by_id(scan_id: int, conn: Connection) -> Dict:
    """Načte scan z databáze podle ID."""
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM reference_front WHERE id = ?", (scan_id,))
    return cursor.fetchone()

# V jiném programu:
from common.db_utils import get_scan_by_id
scan = get_scan_by_id(6901, conn)
```

**Příklad špatně:**
```python
# V programu A:
def get_scan(scan_id):
    cursor.execute("SELECT * FROM reference_front WHERE id = ?", ...)
    
# V programu B:
def load_scan(id):
    cursor.execute("SELECT * FROM reference_front WHERE id = ?", ...)
    
# → Duplicitní kód! Pokud změníš DB schema, musíš upravit na dvou místech.
```

**Enforcement:** Code review při každém commitu - hledat duplicity.

---

### **PRINCIP #2: Root Cause Over Workarounds**

Vždy hledej a řeš skutečnou příčinu problému (root cause), ne pouze symptomy. Žádné fallbacky, žádné shimování, žádné "dočasné" workaroundy.

**Důvod:** Workaroundy vytvářejí technický dluh, skrývají skutečné problémy a časem vedou k neudržitelnému kódu.

**Příklad správně:**
```python
# Problém: Funkce občas vrací None
# ❌ ŠPATNÉ řešení:
result = some_function()
if result is None:
    result = default_value  # Workaround!
    
# ✅ SPRÁVNÉ řešení:
# Zjistit PROČ some_function() vrací None
# Opravit some_function() aby vždy vracela validní hodnotu
# Nebo změnit design aby None byl validní výsledek
```

**Enforcement:** Při code review se ptát "Řešíme příčinu nebo symptom?"

---

### **PRINCIP #3: Backward Compatibility**

Změny v kódu nesmí rozbít existující funkčnost. Pokud měníš signaturu funkce nebo chování modulu, zajisti zpětnou kompatibilitu nebo explicitně migruj všechny callsites.

**Důvod:** Projekt má několik desítek programů. Rozbití zpětné kompatibility může způsobit kaskádové selhání.

**Příklad správně:**
```python
# Stará verze:
def compute_embedding(crop: np.ndarray) -> np.ndarray:
    ...

# Nová verze - přidán parametr:
def compute_embedding(
    crop: np.ndarray,
    model_id: int = 1  # ← Default hodnota pro zpětnou kompatibilitu
) -> np.ndarray:
    ...
    
# Nebo explicitní migrace:
# 1. Vytvoř novou funkci compute_embedding_v2()
# 2. Deprecated warning na starou funkci
# 3. Update všech callsites
# 4. Až všechny updated → remove old function
```

**Enforcement:** Smoke testy po každé změně - batch test na 6800 baseline skenů.

---

### **PRINCIP #4: Token Management Strategy**

Při práci s AI asistenty prioritizuj kvalitu nad kvantitou. Raději spotřebuj polovinu tokenů na pochopení kontextu a implementuj jednu věc správně, než rychle naprogramuj pět věcí špatně.

**Důvod:** Špatná implementace kvůli nedostatku kontextu vytváří technický dluh, který je dražší opravit později.

**Workflow pro komplexní úkoly:**

**Chat #1 - Plánování (50% tokenů na kontext):**
1. Načíst dokumentaci (30% tokenů)
2. Prostudovat souvislosti (20% tokenů)
3. Navrhnout přístup (20% tokenů)
4. Diskuse s člověkem (20% tokenů)
5. Commit: plán + handover (10% tokenů)

**Chat #2 - Implementace části A:**
1. Načíst handover z #1 (10% tokenů)
2. Implementovat část A (60% tokenů)
3. Testovat (20% tokenů)
4. Commit (10% tokenů)

**Chat #3 - Implementace části B:**
- Pokračování...

**Pravidlo "Nikam nespěcháme":**
- Když člověk řekne "tohle je složité", AI nesmí skočit rovnou do programování
- Místo toho: "Pojďme to rozdělit na části", "Nejdřív prostudujeme existující kód"

**Enforcement:** AI sleduje token usage a při 80% varuje.

---

### **PRINCIP #5: Deterministic Behavior**

Stejný vstup musí vždy produkovat stejný výstup. Žádné random inicializace, žádné závislosti na časovém razítku, žádné race conditions.

**Důvod:** Reprodukovatelnost je klíčová pro debugging a vědeckou práci.

**Příklad správně:**
```python
# ✅ SPRÁVNĚ - fixní seed:
torch.manual_seed(12345)
model = create_projection_head(input_dim=2048, output_dim=512)

# ❌ ŠPATNĚ - random:
model = create_projection_head(input_dim=2048, output_dim=512)
# Každé spuštění vrátí jiné embeddings!
```

**Enforcement:** Unit testy musí projít deterministicky 100x za sebou.

---

### **PRINCIP #6: Explicit Over Implicit**

Explicitní kód je lepší než implicitní magie. Raději o pár řádků více, ale jasně čitelných, než zkrácený kód který vyžaduje hluboké znalosti frameworku.

**Příklad správně:**
```python
# ✅ EXPLICITNÍ:
def recognize_stamp_from_array(
    warped_bgr: np.ndarray,
    drawing_id: int,
    env: str = 'dev',
    topk: int = 10,
    debug: bool = False
) -> Dict[str, Any]:
    """
    Rozpozná známku z warped numpy array.
    
    Args:
        warped_bgr: Warped známka (1300x1100x3, BGR, uint8)
        drawing_id: ID kresby (1-6)
        env: Prostředí (dev/prod/sandbox)
        topk: Počet kandidátů k vrácení
        debug: Debug výpisy
        
    Returns:
        Dict s denomination, top_candidates, atd.
    """
    ...
```

**Příklad špatně:**
```python
# ❌ IMPLICITNÍ:
def recognize(*args, **kwargs):
    # Co to bere? Co vrací? 🤷
    ...
```

**Enforcement:** Type hints povinné. Docstringy povinné pro public funkce.

---

### **PRINCIP #7: Test Against Ground Truth**

Každá významná změna v recognition pipeline musí být otestována proti baseline dataset (6800 skenů). Úspěšnost nesmí klesnout pod 96%.

**Důvod:** Regression prevence. Recognition je core funkčnost projektu.

**Příklad:**
```bash
# Po změně v recognize_stamp.py:
python test_batch_recognition.py --baseline 6800

# Expected output:
# Success rate: 96.9% (6653/6800)
# ✅ PASS - no regression
```

**Enforcement:** CI/CD pipeline - automatický batch test při push.

---

### **PRINCIP #8: Comments in Czech, Code in English**

Komentáře v kódu píšeme česky (pro domácí tým), ale názvy proměnných, funkcí a tříd anglicky (univerzální konvence).

**Příklad:**
```python
def compute_embedding(crop: np.ndarray) -> np.ndarray:
    """
    Vypočítá 512D embedding pro daný crop.
    
    Používá ResNet50 s deterministickou projekční hlavou (seed=12345).
    Crop musí být normalizovaný na 1300x1100.
    """
    # Načti model a hlavu
    model = load_resnet50()
    head = load_projection_head(seed=12345)
    
    # Vypočítej embedding
    with torch.no_grad():
        features = model(crop)
        embedding = head(features)
        
    # L2 normalizace
    embedding = embedding / torch.norm(embedding)
    
    return embedding.cpu().numpy()
```

**Důvod:** Anglické názvy = kompatibilita s open-source knihovnami. České komentáře = rychlejší porozumění pro tým.

---

### **PRINCIP #9: Fail Fast, Fail Loud**

Pokud něco nejde jak má, program musí okamžitě selhat s jasnou chybovou hláškou. Žádné tiché selhání, žádné logování warning a pokračování.

**Příklad správně:**
```python
def load_scan(scan_id: int, conn) -> Dict:
    result = conn.execute("SELECT * FROM reference_front WHERE id = ?", (scan_id,)).fetchone()
    
    if result is None:
        raise ValueError(f"Scan {scan_id} not found in database!")
        # ✅ Okamžité selhání s jasnou hláškou
    
    return dict(result)
```

**Příklad špatně:**
```python
def load_scan(scan_id: int, conn) -> Dict:
    result = conn.execute("SELECT * FROM reference_front WHERE id = ?", (scan_id,)).fetchone()
    
    if result is None:
        logging.warning(f"Scan {scan_id} not found")
        return {}  # ❌ Tiché selhání - program pokračuje s prázdným dictem
```

**Enforcement:** Code review - hledat `return None` nebo `return {}` bez exception.

---

### **PRINCIP #10: Paths from paths.json**

Všechny cesty k souborům musí být načítány z `config/paths.json` pomocí `load_config.py`. Žádné hardcoded cesty v kódu.

**Příklad správně:**
```python
from common.load_config import load_config

config = load_config(env='dev')
db_path = config['db_path']  # ✅ Z paths.json
```

**Příklad špatně:**
```python
db_path = "dev/db/hradcany.sqlite"  # ❌ Hardcoded!
```

**Důvod:** Flexibilita - snadné přepínání mezi dev/prod/sandbox. Single source of truth.

---

## 📋 CHECKLIST PRO CODE REVIEW

Při code review kontroluj:

- [ ] Žádné duplicitní funkce/kód
- [ ] Root cause řešen, ne workaround
- [ ] Zpětná kompatibilita zachovaná
- [ ] Type hints přítomné
- [ ] Docstringy u public funkcí
- [ ] Komentáře česky, kód anglicky
- [ ] Žádné hardcoded cesty
- [ ] Fail fast s jasnou chybou
- [ ] Deterministické chování (fixní seedy)
- [ ] Batch test prošel (96%+)

---

**Tento dokument je živý - aktualizuje se když objevíme nové principy nebo anti-patterns.**