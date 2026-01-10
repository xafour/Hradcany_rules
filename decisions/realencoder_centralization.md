# RealEncoder Centralization

**Tags:** `#embedding` `#refactoring` `#single-source-of-truth`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active

---

## 🎯 ROZHODNUTÍ

Rozhodli jsme se přesunout definici třídy `RealEncoder` z `recognize_stamp.py` do centrálního modulu `common/embedding_utils.py`. Tato změna byla provedena 2025-12-31 v rámci přípravy na GT Management implementaci.

RealEncoder je wrapper kolem ResNet50 modelu, který zajišťuje konzistentní výpočet embeddingů - má zabudované preprocessing (BGR → RGB konverze, resize, normalizace), deterministickou projekční hlavu (seed=12345), a L2 normalizaci výstupu. Je to kritická komponenta pro recognition systém, protože musí garantovat že query známka a referenční známky používají naprosto identický encoder - jakýkoliv rozdíl v preprocessing nebo projekční hlavě by vedl k nekonzistentním embeddingům a špatným výsledkům matchingu.

Před touto změnou byla třída RealEncoder definovaná přímo v `recognize_stamp.py`, což fungovalo dokud byl recognition jediný use case. Ale s implementací GT Management jsme potřebovali stejný encoder i v `gt_upload_utils.py` (pro auto-detect denomination). Duplikovat kód by bylo špatné řešení - při jakékoliv změně bychom museli pamatovat na update na dvou místech. Centralizace do `embedding_utils.py` zajišťuje single source of truth.

---

## 🔍 KONTEXT A DŮVODY

### Problém s duplikací

Když jsme začali implementovat UC-1 (Upload & Analyze), narazili jsme na problém: potřebujeme vypočítat embeddings z nahraného skenu a porovnat je s referenčními embeddingy aby jsme mohli navrhnout klasifikaci. To vyžaduje stejný encoder jaký používá `recognize_stamp.py`.

První nápad byl zkopírovat definici RealEncoder z `recognize_stamp.py` do `gt_upload_utils.py`. Ale to by vedlo k duplikaci ~50 řádků kódu a k riziku divergence - co když později změníme preprocessing v jednom souboru a zapomeneme to změnit v druhém? Embeddings by pak nebyly konzistentní.

### RealEncoder vs ResNet50Embed

Je důležité rozlišovat mezi `ResNet50Embed` (raw PyTorch model) a `RealEncoder` (wrapper s preprocessing). ResNet50Embed očekává PyTorch tensor na vstupu a vrací raw 2048-dimenzionální vektor. RealEncoder očekává numpy BGR array (jak ho vrací OpenCV) a vrací L2-normalizovaný 512-dimenzionální vektor (po průchodu projekční hlavou).

Kdyby jsme v `gt_upload_utils.py` použili přímo ResNet50Embed místo RealEncoder, museli bychom duplikovat všechen preprocessing kód - BGR→RGB konverzi, resize na 224×224, normalizaci, konverzi na tensor, načtení projekční hlavy, forward pass, L2 normalizaci. To by bylo ještě horší než duplikovat celý RealEncoder.

### Single Source of Truth

Principem single source of truth je: každý kus logiky by měl být definovaný na JEDNOM místě. Když potřebujeme tu logiku na více místech, importujeme ji, ne kopírujeme.

Centralizace RealEncoder do `embedding_utils.py` znamená: (1) definice existuje na jednom místě, (2) změny se automaticky projeví všude kde se používá, (3) nemůže dojít k divergenci mezi instancemi, (4) snazší testování - testujeme jeden modul místo více kopií.

---

## 💡 DŮSLEDKY

### Backward Compatibility

Přesun RealEncoder do `embedding_utils.py` nijak nenarušil existující funkčnost `recognize_stamp.py`. Změnili jsme pouze kde je RealEncoder definovaný, ne jak funguje. `recognize_stamp.py` teď importuje RealEncoder místo toho aby ho definoval lokálně:

```python
# Před (recognize_stamp.py obsahoval definici):
class RealEncoder:
    def __init__(self, model_id, env):
        # ... 50 řádků kódu ...

# Po (recognize_stamp.py importuje):
from common.embedding_utils import RealEncoder
```

Všechny testy `recognize_stamp.py` stále prošly, baseline validation (6,800 skenů) dala stejné výsledky. To potvrzuje že přesun byl clean refactoring bez změny chování.

### Nová funkcionalita v GT Management

`gt_upload_utils.py` teď může používat RealEncoder pro auto-detect denomination:

```python
from common.embedding_utils import RealEncoder

# V UC-1 workflow:
encoder = RealEncoder(model_id=model_id, env='dev')
query_embedding = encoder(warped_crop)  # numpy BGR → 512D L2-normalized
# ... porovnej s reference embeddings ...
```

Protože je to STEJNÝ encoder jako používá `recognize_stamp.py`, máme garantováno že embeddings jsou konzistentní. Query embedding z UC-1 lze přímo porovnat s reference embeddings v databázi.

### Code Elimination

Refactoring eliminoval ~50 řádků duplicitního kódu. `recognize_stamp.py` se zmenšil (import místo definice), `gt_upload_utils.py` nepotřebuje duplikovat kód, a `embedding_utils.py` se stal centrálním místem pro vše co souvisí s embeddingy.

---

## 🔧 TECHNICKÉ DETAILY

### Co RealEncoder dělá

```python
class RealEncoder:
    def __init__(self, model_id: int, env: str):
        # 1. Načte ResNet50 backbone
        self.backbone = load_resnet50()
        
        # 2. Načte deterministickou projekční hlavu (2048→512)
        self.head = load_projection_head(
            model_id=model_id,
            seed=12345  # FIXED seed pro determinismus
        )
        
    def __call__(self, crop_bgr: np.ndarray) -> np.ndarray:
        """
        Preprocessing + forward pass + L2 normalization.
        
        Input: numpy array (H, W, 3), BGR, uint8
        Output: numpy array (512,), float32, L2-normalized
        """
        # 1. BGR → RGB
        crop_rgb = cv2.cvtColor(crop_bgr, cv2.COLOR_BGR2RGB)
        
        # 2. Resize → 224×224
        crop_resized = cv2.resize(crop_rgb, (224, 224))
        
        # 3. Normalize [0,1]
        crop_normalized = crop_resized / 255.0
        
        # 4. To tensor
        tensor = torch.from_numpy(crop_normalized).permute(2,0,1).unsqueeze(0)
        
        # 5. Forward pass: ResNet50 → 2048D
        with torch.no_grad():
            features = self.backbone(tensor)  # (1, 2048)
        
        # 6. Projection head: 2048D → 512D
        embedding = self.head(features)  # (1, 512)
        
        # 7. L2 normalization
        embedding_normalized = F.normalize(embedding, p=2, dim=1)
        
        # 8. To numpy
        return embedding_normalized.cpu().numpy()[0]  # (512,)
```

Všimni si že preprocessing je SOUČÁSTÍ RealEncoder. Když voláme `encoder(crop)`, dostáváme už kompletně zpracovaný embedding. Nemusíme pamatovat na všechny preprocessing kroky - RealEncoder to dělá automaticky.

### Proč deterministická projekční hlava

Projekční hlava (2048→512 linear layer) je inicializovaná s FIXNÍM seedem (12345). To zajišťuje že při každém načtení modelu dostaneme identickou hlavu - stejné váhy, stejný bias.

Důvod: Referenční embeddings v databázi byly vypočítané s konkrétní projekční hlavou. Pokud bychom query embeddings počítali s JINOU hlavou (náhodně inicializovanou), nebyly by porovnatelné. Fixed seed zajišťuje reprodukovatelnost.

---

## 🔗 SOUVISLOSTI

**Používáno v:**
- `recognize_stamp.py` v3.1.1 - Recognition pipeline
- `gt_upload_utils.py` v1.6.0 - UC-1 auto-detect
- `compute_reference_embeddings.py` - Baseline embeddings computation

**Technická dokumentace:**
- [base/pipeline_recognition.md](../base/pipeline_recognition.md) - Jak recognition používá RealEncoder

**Další rozhodnutí:**
- [decisions/uc1_workflow.md](./uc1_workflow.md) - Jak UC-1 používá RealEncoder pro auto-detect

**Implementace:**
- `common/embedding_utils.py` v1.3.0 - Centrální definice
- Migration: 2025-12-31, verified by baseline tests

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Status:** ✅ ACTIVE

---
