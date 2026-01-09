# ARCHITEKTURA PROJEKTU HRADČANY

**Verze:** 0.9.1  
**Datum:** 2026-01-09  

---

## 🎯 ZÁKLADNÍ FILOZOFIE

### **Klíčové pravidlo:**

Cílem je systém, který bude publikovaný a internetu. Vývoj a testování probíhá lokálně. Již při vývoji bereme do úvahy, že publikované adresářové struktury budou oddělené od adresářů koncových uživatelů.

## 📁 ADRESÁŘOVÁ STRUKTURA

```
~/ProjektHradcany/
├── shared/                           # 🔒 CÍL: nebude sdílené
│   ├── models/                      # Production ML modely (přesunout do sdíleného)
│   │   ├── model_production/        # Stable YOLO model
│   │   │   └── weights/best.pt
│   │   └── embed_head/              # Embedding projection heads
│   ├── data_input/                  # Baseline reference data (READ-ONLY!)
│   │   └── hradcany/                # 6800 reference skenů (ZP_001-100, TD I-II)
│   └── config/                      # **přesunout do sdíleného**
│       └── paths.json               # SINGLE SOURCE OF TRUTH
│
├── common/                           # 🔧 SDÍLENÝ KÓD - zvážit přesunutí do <ENV> 
│   ├── load_config.py               # Path management (v1.8+)
│   ├── yolo_utils.py                # YOLO inference wrapper
│   ├── gt_utils.py                  # GT management functions
│   ├── embedding_utils.py           # Embedding computations
│   ├── parse_utils.py               # JSON parsers
│   └── detector.py                  # Frame detection
│
├── tools/                            # 🛠️ UTILITY SKRIPTY (CLI tools) - 🔒 CÍL: nebude sdílené
│   ├── gt_file_manager.py           # GT workflow (--env required)
│   ├── web_scraper.py               # Auction data scraper (--env required)
│   ├── fill_missing_inference_frames.py
│   ├── backup_hradcany_timestamp.sh
│   └── git_commit.sh
│
├── dev/                              # 🧪 DEVELOPMENT ENVIRONMENT 🔒 CÍL: simuluje sdílené PROD
│   ├── code/                        # Dev-specific scripts
│   │   ├── recognize_stamp.py       # Main inference
│   │   ├── compute_reference_embeddings.py
│   │   ├── augment_with_obb.py      # Training pipeline
│   │   └── export_to_yolo_obb.py
│   ├── data/                        # ⚠️ DYNAMICKÁ DATA (git ignored)
│   │   ├── gt/                      # Ground Truth (roste!)
│   │   │   ├── 5h3/TD1/ZP001_*.jpg
│   │   │   ├── 500h/TD1/ZP042_*.jpg
│   │   │   └── ...
│   │   ├── pending/                 # Čeká na expert verifikaci
│   │   │   └── 2025-11/*.jpg
│   │   ├── scraper/                 # Aukční data (dočasné)
│   │   ├── outputs/                 # Inference results
│   │   ├── ramec_kresby_training_obb/  # YOLO training experiments
│   │   ├── augment_with_obb/        # Augmentation workspace
│   │   └── anotace_kresby_all/      # Annotation workspace
│   ├── db/                          # SQLite (git ignored, ~50 MB)
│   │   └── hradcany.sqlite
│   ├── models/                      # Symlink → shared/models/
│   └── logs/                        # Runtime logs
│
├── prod/                            # 🚀 PRODUCTION ENVIRONMENT 🔒 CÍL: bude sdílené
│   ├── code/                        # Stable production scripts
│   ├── data/
│   │   └── gt/                      # FINAL verified GT only
│   ├── db/                          # Production database
│   └── logs/
│
└── sandbox/                         # CÍL: simuluje sdílené PROD pro eyperimenty
    ├── code/                        # Throwaway experiments
    ├── data/                        # Volatile test data
    └── db/                          # Test database
```

---

## 🔄 DATA FLOW & LIFECYCLE

### **1. ML Model Lifecycle**

```
┌─────────────────────────────────────────────────────────────┐
│ YOLO MODEL EVOLUTION                                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  dev/data/ramec_kresby_training_obb/                        │
│  └── training_run_2025-11-28/                               │
│      ├── train/                                              │
│      ├── val/                                                │
│      └── weights/                                            │
│          └── best.pt  ← Experimentální model                │
│                                                              │
│              ↓ Test & validate                               │
│                                                              │
│  shared/models/model_production/                            │
│  └── weights/                                                │
│      └── best.pt  ← STABLE, používaný v pipeline            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Strategie:**
- ✅ Experimentuj v `dev/data/ramec_kresby_training_obb/`
- ✅ Validuj na dev GT data
- ✅ Když je stabilní → přesuň do `shared/models/`
- ✅ Pipeline používá jen `shared/models/` (production)

### **2. Ground Truth Lifecycle**

```
┌─────────────────────────────────────────────────────────────┐
│ GT DATA WORKFLOW                                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. ACQUISITION                                              │
│     tools/web_scraper.py --env dev                          │
│     → dev/data/scraper/2025-11-28_auction/                  │
│                                                              │
│  2. IMPORT TO PENDING                                        │
│     tools/gt_file_manager.py --env dev --action add         │
│     → dev/data/pending/2025-11/scan_XXX.jpg                 │
│     → dev/data/pending/2025-11/scan_XXX.jpg.notes.json      │
│                                                              │
│  3. ANALYZE                                                  │
│     dev/code/recognize_stamp.py --env dev                   │
│     → Použije: shared/data_input/hradcany/ (baseline)       │
│     → Použije: shared/models/model_production/ (YOLO)       │
│     → Zapíše: dev/db/hradcany.sqlite                        │
│                                                              │
│  4. VERIFY                                                   │
│     tools/gt_file_manager.py --env dev --action verify      │
│     → dev/data/gt/500h/TD1/ZP042_SHA256.jpg                 │
│     → DB: confirmed=1, is_reference=1                       │
│                                                              │
│  5. PRODUCTION PROMOTION (manual)                           │
│     cp dev/data/gt/500h/TD1/ZP042_*.jpg \                   │
│        prod/data/gt/500h/TD1/                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### **3. Baseline Reference Data**

```
┌─────────────────────────────────────────────────────────────┐
│ BASELINE DATA (READ-ONLY, SHARED)                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  shared/data_input/hradcany/                                │
│  ├── 5h3/TD1/ZP_001_abc123.jpg    ← 6800 reference skenů   │
│  ├── 5h3/TD2/...                                            │
│  ├── 500h/TD1/...                                           │
│  └── ...                                                     │
│                                                              │
│  Použití:                                                    │
│  - compute_reference_embeddings.py (výpočet embeddingů)     │
│  - recognize_stamp.py (cosine similarity matching)          │
│                                                              │
│  ⚠️  NIKDY SE NEMĚNÍ - společné pro všechny ENV!            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Proč shared?**
- ✅ READ-ONLY (nikdy se nemění)
- ✅ Velikost ~2-3 GB (zbytečná duplikace)
- ✅ Všechny ENV testují proti stejným baseline
- ✅ Symlinky možné pro convenience

---

## 🗂️ PATHS.JSON - SINGLE SOURCE OF TRUTH

### **Kompletní struktura:**

```json
{
  "shared": {
    "root": "shared",
    "models": "shared/models",
    "baseline": "shared/data_input/hradcany",
    "config": "shared/config"
  },
  
  "local": {
    "root": "data",
    
    "gt": "data/gt",
    "pending": "data/pending",
    "scraper": "data/scraper",
    "outputs": "data/outputs",
    "experiments": "data/experiments",
    
    "db": "db",
    "logs": "logs",
    "models": "models",
    
    "augment_with_obb": "data/augment_with_obb",
    "anotace_images": "data/anotace_kresby_all",
    "ramec_training": "data/ramec_kresby_training_obb"
  }
}
```

### **Jak používat:**

```python
# ✅ CORRECT - auto-detekce ENV z cesty
from common.load_config import get_paths

# Program automaticky detekuje dev/prod/sandbox z __file__
paths = get_paths(__file__)

# Baseline (shared)
baseline_dir = Path(paths["abs"]["baseline"])

# GT data (per-ENV)
gt_dir = Path(paths["abs"]["gt"])

# Scraper output (per-ENV)
scraper_dir = Path(paths["abs"]["scraper"])

# Database (per-ENV)
db_path = Path(paths["abs"]["db"]) / "hradcany.sqlite"

# ❌ WRONG - hardcoded
gt_dir = Path.home() / "ProjektHradcany/dev/data/gt"  # NE!
```

---

## 🔧 COMMON/ - SDÍLENÝ KÓD

### **Co patří do common/:**

#### ✅ **1. Infrastructure (parametrizované)**
```python
# load_config.py
def get_paths(env_or_path=None, ...):
    """
    SINGLE SOURCE OF TRUTH pro cesty.
    
    Podporuje:
    - get_paths(__file__)       # Auto-detekce z cesty (PRIMÁRNÍ)
    - get_paths(env="dev")      # Explicitní ENV (fallback)
    - get_paths("dev")          # Zkrácený styl (fallback)
    """
```

**POZNÁMKA:** Funkce jako `get_gt_paths()`, `get_scraper_paths()` jsou DEPRECATED!  
→ Použij přímo `get_paths(__file__)` a extrahuj potřebné klíče.

#### ✅ **2. Pure Utilities (žádné I/O)**
```python
# yolo_utils.py
def detect_frame_yolo(image_path, model_path, conf_threshold=0.7):
    """Čistá YOLO inference, vrací quad"""

# embedding_utils.py
def compute_embedding(image, model):
    """Čistý embedding výpočet"""
    
# parse_utils.py
def parse_auction_html(html_content):
    """Čistý parsing, žádné I/O"""
```

#### ✅ **3. Shared Business Logic**
```python
# detector.py
def detect_and_warp(image_bgr, model_path):
    """Detekce + warp, používá yolo_utils"""
```

### **Co NEPATŘÍ do common/:**

#### ❌ **1. Hardcoded Paths**
```python
# ❌ BAD
DB_PATH = Path.home() / "ProjektHradcany/dev/db/hradcany.sqlite"

# ✅ GOOD - přijmi jako parametr
def process(env):
    paths = get_paths(env=env)
    db_path = Path(paths["abs"]["db"]) / "hradcany.sqlite"
```

#### ❌ **2. ENV-Specific Logic**
```python
# ❌ BAD - co když chci sandbox?
if "dev" in str(Path.cwd()):
    do_something()

# ✅ GOOD - explicitní parametr
def process(env, debug=False):
    if debug:
        do_something()
```

---

## 🛠️ TOOLS/ - CLI UTILITY SKRIPTY

### **Pravidla:**

1. **Auto-detekce ENV z cesty:**
   ```bash
   # Program sám detekuje ENV podle toho, odkud je spuštěn
   cd ~/ProjektHradcany/dev
   python tools/gt_file_manager.py --action add ...
   # → auto-detekuje "dev"
   
   cd ~/ProjektHradcany/prod  
   python tools/gt_file_manager.py --action add ...
   # → auto-detekuje "prod"
   ```

2. **ŽÁDNÉ delegování na utility funkce:**
   ```python
   # ❌ ŠPATNĚ (zbytečné delegování):
   from gt_utils import get_gt_paths
   gt_paths = get_gt_paths(env)
   
   # ✅ SPRÁVNĚ (přímé použití load_config):
   from load_config import get_paths
   paths = get_paths(__file__)
   gt_dir = Path(paths["abs"]["gt"])
   ```

3. **Business logic v common/, wrappers v tools/:**
   ```python
   # tools/gt_file_manager.py
   from common.gt_utils import add_to_pending, verify_scan
   from common.load_config import get_paths
   
   paths = get_paths(__file__)  # auto-detekce
   
   if args.action == "add":
       add_to_pending(args.file, paths, note=args.note)
   ```

---

## 📊 VELIKOSTI & STORAGE

```
SHARED (sdílené mezi všemi ENV):
  shared/models/              ~500 MB    (ML modely)
  shared/data_input/hradcany/ ~2-3 GB    (6800 baseline skenů)
  shared/config/              ~1 KB      (paths.json)
  CELKEM:                     ~2.5-3.5 GB

PER-ENV (každé prostředí vlastní):
  {env}/data/gt/              ~100-500 MB (roste postupně)
  {env}/data/pending/         ~10-50 MB   (dočasné)
  {env}/data/scraper/         ~50-200 MB  (dočasné)
  {env}/data/outputs/         ~10-50 MB
  {env}/db/                   ~50 MB      (SQLite)
  CELKEM PER ENV:             ~220-850 MB

CODE (společný):
  common/                     ~100 KB     (Python moduly)
  tools/                      ~50 KB      (CLI skripty)

TRAINING (DEV only):
  dev/data/ramec_training/    ~1-2 GB     (YOLO experiments)
```

**➡️ Duplikace baseline = 3 GB × 3 ENV = 9 GB zbytečně!**  
**➡️ Shared strategie = úspora ~6 GB**

---

## 🔐 GIT IGNORE STRATEGIE

```gitignore
# Database (dynamická data)
*/db/*.sqlite
*/db/*.sqlite-*

# ENV-specific data
*/data/gt/
*/data/pending/
*/data/scraper/
*/data/outputs/
*/data/experiments/

# Training artifacts
*/data/ramec_kresby_training_obb/*/weights/
*/data/ramec_kresby_training_obb/*/runs/

# Python
__pycache__/
*.pyc
*.pyo
.pytest_cache/

# Virtual environments
venv/
.venv/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Logs
*.log
```

**CO VERZOVAT:**
- ✅ `common/` (všechny .py)
- ✅ `tools/` (všechny .py, .sh)
- ✅ `{env}/code/` (všechny .py)
- ✅ `shared/config/paths.json`
- ✅ Dokumentace (*.md)

**CO NEVERZOVAT:**
- ❌ `shared/models/` (velké binárky, přes Git LFS nebo S3)
- ❌ `shared/data_input/` (baseline data, přes zálohy)
- ❌ `{env}/data/` (dynamická data)
- ❌ `{env}/db/` (databáze)

---

## 🚀 USE CASE PŘÍKLADY

### **Use Case 1: Nový expert přidává známku**

```bash
# 1. Stáhnout z aukce
cd ~/ProjektHradcany/dev
python tools/web_scraper.py --url https://auction.com/lot123

# → Auto-detekuje ENV="dev"
# → Uloží do: dev/data/scraper/2025-11-28_auction/

# 2. Import do pending
python tools/gt_file_manager.py --action add \
    --file data/scraper/.../scan.jpg \
    --note "Burda auction, lot 123" \
    --source "Burda"

# → Auto-detekuje ENV="dev"
# → Uloží do: dev/data/pending/2025-11/scan_XXX.jpg
# → Vytvoří: dev/data/pending/2025-11/scan_XXX.jpg.notes.json

# 3. Analyze
python dev/code/recognize_stamp.py \
    --input data/pending/2025-11/scan_XXX.jpg \
    --denomination 500h

# → Auto-detekuje ENV="dev"
# → Použije baseline: shared/data_input/hradcany/
# → Použije model: shared/models/model_production/
# → Zapíše: dev/db/hradcany.sqlite (scan_id, embeddingy)
# → Auto-load notes z .notes.json

# 4. Review
python tools/gt_file_manager.py --action list-pending

# 5. Verify
python tools/gt_file_manager.py --action verify \
    --scan-id 6902 --plate 1 --zp 42

# → Přesune do: dev/data/gt/500h/TD1/ZP042_SHA256.jpg
# → DB: confirmed=1, is_reference=1
```

### **Use Case 2: Trénink nového YOLO modelu**

```bash
# 1. Prepare training data
python dev/code/export_to_yolo_obb.py --env dev

# → Použije: dev/data/augment_with_obb/
# → Vytvoří: dev/data/augment_with_obb/labels/*.txt

# 2. Augment
python dev/code/augment_with_obb.py --env dev

# → Vytvoří augmentované varianty

# 3. Train
cd dev/data/ramec_kresby_training_obb/
yolo obb train data=dataset.yaml ...

# → Výstup: dev/data/ramec_kresby_training_obb/run_2025-11-28/

# 4. Validate
python dev/code/recognize_stamp.py --env dev \
    --model dev/data/ramec_kresby_training_obb/run_2025-11-28/weights/best.pt \
    --input test_images/

# 5. Promote to production (manual)
cp dev/data/ramec_kresby_training_obb/run_2025-11-28/weights/best.pt \
   shared/models/model_production/weights/best_2025-11-28.pt
```

### **Use Case 3: Compute reference embeddings**

```bash
# Výpočet embeddingů pro baseline
python dev/code/compute_reference_embeddings.py --env dev \
    --model-tag best.pt \
    --model-sha 2f9186b896fb... \
    --where-crops "verified=1 AND is_required=1"

# → Čte: shared/data_input/hradcany/ (baseline)
# → Zapisuje: dev/db/hradcany.sqlite (embeddingy)
# → Ukládá patches: dev/data/outputs/reference_patches/ (debug)
```

---

## 🔄 MIGRATION CHECKLIST

### **Fáze 1: Critical Fixes** ✅ READY
- [ ] Fix `load_config.py` v1.8 (backward compatible)
- [ ] Fix `gt_utils.py` (remove data/data_input)
- [ ] Update `paths.json` (add missing keys)
- [ ] Fix `gt_batch_import.py` (use gt_utils paths)
- [ ] Fix `web_scraper.py` (add --env, use paths.json)
- [ ] Fix `compute_reference_embeddings.py` (use paths.json)
- [ ] Smoke test pipeline

### **Fáze 2: Directory Reorganization**
- [ ] Move `shared/gt/` → `dev/data/gt/`
- [ ] Move `shared/pending/` → `dev/data/pending/`
- [ ] Verify `shared/data_input/hradcany/` (baseline OK)
- [ ] Update dokumentace
- [ ] Git commit

### **Fáze 3: Optional Refactoring** (budoucnost)
- [ ] Migrate PRE-REFACTOR scripts
- [ ] Add --env to shell scripts
- [ ] Remove hardcoded paths everywhere

---

## 📝 DEVELOPMENT GUIDELINES

### **1. Při psaní nového skriptu:**

```python
# ✅ TEMPLATE pro nový skript
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Název: my_script.py
Verze: 1.0.0
Datum: 2025-11-28

Popis: Co skript dělá
"""
import argparse
from pathlib import Path
import sys

sys.path.append(str(Path(__file__).resolve().parents[2]))
from common.load_config import get_paths

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--debug", action="store_true", help="Debug režim")
    # ... další argumenty
    args = ap.parse_args()
    
    # Načti paths (auto-detekce ENV z __file__)
    paths = get_paths(__file__)
    
    # Použij paths.json klíče
    db_path = Path(paths["abs"]["db"]) / "hradcany.sqlite"
    gt_dir = Path(paths["abs"]["gt"])
    
    if args.debug:
        print(f"[DEBUG] ENV: {paths['env']}")
        print(f"[DEBUG] DB: {db_path}")
        print(f"[DEBUG] GT: {gt_dir}")
    
    # ... business logic

if __name__ == "__main__":
    main()
```

### **2. Před commitem:**

```bash
# 1. Smoke test v příslušném ENV
python my_script.py --env dev --input test.jpg

# 2. Check že nepoužíváš hardcoded paths
rg "Path.home\(\)" my_script.py  # Mělo by být prázdné!
rg "ProjektHradcany" my_script.py  # Mělo být prázdné!

# 3. Update dokumentace
vim PROJECT_STATUS.md

# 4. Git commit
bash tools/git_commit.sh "feat: add my_script.py"
```

### **3. Při refactoringu:**

```bash
# 1. Backup
cp common/old_module.py common/old_module_backup.py

# 2. Smoke test PŘED
python dev/code/recognize_stamp.py --env dev --input test.jpg

# 3. Refactor

# 4. Smoke test PO
python dev/code/recognize_stamp.py --env dev --input test.jpg

# 5. Pokud OK → commit, jinak rollback
```

---

## 🎯 SUCCESS METRICS

### **Kontrola že architektura funguje:**

```bash
# ✅ Žádné hardcoded paths
rg "Path.home\(\).*ProjektHradcany" --type py | wc -l
# → Mělo by být 0!

# ✅ Všechny programy mají --env
rg "argparse" tools/*.py | rg "env" | wc -l
# → Mělo by být N (počet toolů)

# ✅ Shared baseline je READ-ONLY
ls -l shared/data_input/hradcany/ | wc -l
# → Mělo by být 6800 (nikdy se nemění!)

# ✅ ENV separace funguje
ls dev/data/gt/ prod/data/gt/
# → Měly by být RŮZNÉ soubory!

# ✅ Pipeline funguje
bash tools/run_pipeline_final.sh DEV 1 1
# → Mělo by projít BEZ CHYB!
```

---

## 📚 SOUVISEJÍCÍ DOKUMENTY

- `PROJECT_STATUS.md` - Aktuální stav projektu
- `CUSTOM_INSTRUCTIONS.md` - Claude instructions
- `TASK-999_FINAL_REPORT.md` - Paths consolidation audit
- `README.md` - Getting started guide

---

**Poslední update:** 2025-11-28  
**Autor:** Milan Bojanovský  
**Status:** ✅ APPROVED - READY TO IMPLEMENT
