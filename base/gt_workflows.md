# GT Workflows - Implementation Guide

**Tags:** `#gt-management` `#workflows` `#wf1` `#wf2` `#wf3` `#wf4`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active

---

## 🎯 ÚČEL

Tento dokument popisuje čtyři hlavní workflows v GT Management systému a poskytuje implementační detaily pro každý z nich. Workflows jsou high-level procesy které sdružují související Use Cases do ucelených činností podle typu aktivity a uživatelské role.

Každý workflow má jasně definovaný začátek, konec, a odpovědi na otázky: kdo ho spouští (user / expert / system), co je vstup, co je výstup, a jaké Use Cases jsou součástí workflow. Tento dokument slouží jako referenční příručka pro implementaci - popisuje JAK mají workflows fungovat z technického pohledu.

---

## WF-1: USER UPLOAD & ANALYZE

### Účel

WF-1 řeší základní cestu nového skenu do systému od nahrání po potvrzení klasifikace. Je navržen jako dvoufázový proces který odděluje automatickou analýzu (UC-1) od potvrzení metadata (UC-2), což zajišťuje že do Ground Truth se dostanou pouze ověřené známky.

### Workflow Steps

```
[User] → UC-1: Upload & Analyze
         ↓
      [System analyzuje sken]
         ↓
      [Navrhne classification]
         ↓
      [Status: PENDING]
         ↓
[User/Expert] → UC-2: Confirm Classification
         ↓
      [Update metadata]
         ↓
      [Compute embeddings]
         ↓
      [Check auto-approval]
         ↓
      [Status: VERIFIED nebo PENDING GT APPROVAL]
```

### UC-1: Upload & Analyze

**Kdo:** Běžný uživatel nebo expert  
**Vstup:** Front scan (required), volitelně back/cert/block  
**Výstup:** scan_id, auto-detected denomination, confidence, TOP-5 matches

**Kroky (11 total):**
1. YOLO detekuje rámeček (quad)
2. Warp na 1300×1100 (dočasně pro zpracování)
3. Compute SHA256 z ORIGINÁLU
4. Check duplicita podle SHA256
5. Save files do SHA-based storage (ORIGINÁL!)
6. DB INSERT: reference_front (metadata=NULL, confirmed=0)
7. DB INSERT: user_uploads (ownership tracking)
8. DB INSERT: inference_frames (YOLO cache)
9. DB INSERT: scan_supplementary_files (pokud jsou)
10. Auto-detect denomination (LIVE, neukládá se)
11. Audit log

**Status po UC-1:** Sken je PENDING (confirmed=0, metadata=NULL)

**Implementace:** `common/gt_upload_utils.py` v1.6.0  
**Známý bug:** Ukládá warp místo originálu (viz `tasks/bug_uc1_file_storage.md`)

### UC-2: Confirm Classification

**Kdo:** Uživatel který uploadoval, nebo expert  
**Vstup:** scan_id, confirmed classification (stamp_type_id, plate_id, zp_no)  
**Výstup:** scan_id, embeddings computed, auto_approved flag

**Kroky:**
1. Ověř že user má oprávnění (vlastní sken nebo je expert)
2. UPDATE reference_front: vyplň metadata, confirmed=1
3. Načti drawing_crops pro danou drawing_id
4. Načti originální sken ze storage (path reconstruction z SHA256)
5. Warp na 1300×1100 (dočasně)
6. Pro každý required crop:
   - Vyřízni oblast
   - Compute embedding pomocí RealEncoder
   - DB INSERT: reference_embeddings (is_reference=0, pending GT approval)
7. Check auto-approval conditions (certifikát + HIGH confidence + OCR match)
8. Pokud auto-approved:
   - UPDATE: embeddings is_reference=1
   - Sken je okamžitě v GT
9. Pokud ne:
   - Embeddings zůstávají is_reference=0
   - Čeká na expert approval (UC-5)
10. Audit log

**Status po UC-2:** Sken je VERIFIED (confirmed=1, metadata vyplněná), embeddings čekají na GT approval

---

## WF-2: EXPERT REVIEW & VERIFICATION

### Účel

WF-2 řeší pokročilé činnosti experta při správě Ground Truth - schvalování pending skenů, odmítání špatných skenů, a pravidelnou kontrolu kvality GT. Expert má odpovědnost za kvalitu Ground Truth, což je kritické pro přesnost recognition systému.

### UC-5: Expert Approval (TODO)

**Kdo:** Expert  
**Vstup:** scan_id (z pending queue)  
**Výstup:** Sken schválen do GT

**Varianty:**
- **Manual approval:** Expert zkontroluje jeden sken a schválí
- **Auto-approval:** Systém automaticky schválí skeny které splňují kritéria (certifikát + HIGH confidence + OCR match)
- **Batch approval:** Expert vybere více skenů a schválí hromadně

**Kroky (manual):**
1. Expert si prohlédne sken (front, volitelně back)
2. Zkontroluje navrhovanou klasifikaci (stamp_type_id, plate_id, zp_no)
3. Pokud souhlasí:
   - UPDATE: reference_embeddings is_reference=1 (aktivní v GT)
   - Sken je součástí GT, používá se pro matching
4. Pokud nesouhlasí:
   - Demotuje zpět do pending (UC-2 s jinou klasifikací)
   - Nebo odmítne (UC-6)
5. Audit log

**Status po UC-5:** Sken je v GROUND TRUTH (embeddings is_reference=1)

### UC-6: Expert Rejection (TODO)

**Kdo:** Expert  
**Vstup:** scan_id, důvod odmítnutí  
**Výstup:** Sken rejected nebo reclassified

**Varianty:**
- **Soft delete:** confirmed=-1, embeddings se smažou, sken je "rejected" ale zůstává v DB pro audit
- **Reclassify:** Expert opraví metadata a schválí (UC-5)

### UC-7: Quality Review (TODO)

**Kdo:** Expert nebo system (automatická periodic review)  
**Vstup:** scan_id nebo batch skenů z GT  
**Výstup:** Skeny potvrzené nebo flagované jako suspect

**Aktivní kroky:**
- **Flag suspect:** Expert označí sken jako podezřelý (suspect_flag=1, suspect_reason)
- **Demote to pending:** Expert demotuje sken z GT zpět do pending (is_reference=0)
- **Reclassify:** Expert opraví metadata
- **Remove from GT:** Expert odstraní sken z GT (confirmed=-1)
- **Clear suspect flag:** Expert potvrdí že sken je OK (suspect_flag=0)

**Automatická periodic review:**
- System každých 30 dní automaticky flaguje GT skeny pro review
- Expert je zkontroluje a buď potvrdí nebo demotuje

---

## WF-3: USER MANAGEMENT

### Účel

WF-3 řeší jak běžný uživatel spravuje své vlastní uploady. Každý uživatel vidí POUZE svoje skeny (privacy-first design), může je filtrovat podle stavu, a mazat ze svého view.

### UC-3: View My Uploads (TODO)

**Kdo:** Uživatel  
**Vstup:** Filter (pending / verified / rejected / all)  
**Výstup:** Seznam skenů které uživatel uploadoval

**SQL query:**
```sql
SELECT rf.*, uu.uploaded_at, uu.upload_source
FROM reference_front rf
JOIN user_uploads uu ON rf.id = uu.scan_id
WHERE uu.uploaded_by = :current_user
  AND rf.confirmed = :filter  -- 0=pending, 1=verified, -1=rejected
ORDER BY uu.uploaded_at DESC;
```

**Privacy:** Uživatel NIKDY nevidí cizí uploady. Dotaz filtruje podle `uploaded_by`.

### UC-4: Batch Import (TODO)

**Kdo:** Uživatel (pokročilý) nebo system  
**Vstup:** Multiple files (API upload, CSV list, directory)  
**Výstup:** Batch of scan_ids

**Varianty:**
- **API endpoint:** POST /api/upload/batch s multiple files
- **CSV import:** CSV se seznamem paths + metadata
- **Streaming:** Upload velkých dávek s progress reporting

### UC-17: Delete from View (TODO)

**Kdo:** Uživatel  
**Vstup:** scan_id  
**Výstup:** Sken smazán z user view

**Kroky:**
1. DELETE FROM user_uploads WHERE scan_id=:scan_id AND uploaded_by=:current_user
2. Check: Je user poslední vlastník tohoto skenu?
3. Pokud ano:
   - Pokud sken je PENDING (confirmed=0): DELETE from reference_front (cascade)
   - Pokud sken je VERIFIED/GT: Nelze smazat (varování)
4. Pokud ne:
   - Smaž jen záznam v user_uploads, sken zůstává pro ostatní
5. Audit log

**Multi-user safe:** Soubor se smaže ze storage až když ho smažou všichni vlastníci.

---

## WF-4: EXPERT QUEUES

### Účel

WF-4 definuje CO expert vidí když se přihlásí do systému. Expert má přístup ke třem frontám, ne k celému Ground Truth katalogu.

### Queue 1: Pending Queue

**Co obsahuje:** Všechny skeny se stavem confirmed=1 (verified) ale embeddings is_reference=0 (čekají na GT approval)

**SQL:**
```sql
SELECT rf.*
FROM reference_front rf
WHERE rf.confirmed = 1
  AND EXISTS (
    SELECT 1 FROM reference_embeddings re
    WHERE re.scan_id = rf.id AND re.is_reference = 0
  )
ORDER BY rf.id ASC;
```

**Akce:** Expert může schválit (UC-5) nebo odmítnout (UC-6)

### Queue 2: Suspect Queue

**Co obsahuje:** Všechny skeny označené jako podezřelé (suspect_flag=1)

**SQL:**
```sql
SELECT rf.*
FROM reference_front rf
WHERE rf.suspect_flag = 1
ORDER BY rf.id ASC;
```

**Akce:** Expert může clear flag, demotovat do pending, nebo odstranit z GT (UC-7)

### Queue 3: Own Uploads

**Co obsahuje:** Všechny skeny které expert sám uploadoval

**SQL:**
```sql
SELECT rf.*, uu.uploaded_at
FROM reference_front rf
JOIN user_uploads uu ON rf.id = uu.scan_id
WHERE uu.uploaded_by = :expert_user
ORDER BY uu.uploaded_at DESC;
```

**Akce:** Expert vidí svoje uploady stejně jako běžný uživatel (UC-3)

### Co expert NEVIDÍ

Expert NEVIDÍ:
- Kompletní GT katalog (stránkovatelný seznam všech známek)
- Cizí uploady které nejsou v pending/suspect queue
- Statistiky GT (kolik známek per denomination)

**Důvod:** Ground Truth je interní know-how, ne browsable katalog. Expert potřebuje vidět co vyžaduje jeho pozornost, ne všechno.

---

## 🔗 SOUVISLOSTI

**Celkový koncept:**
- [decisions/gt_management_use_cases.md](../decisions/gt_management_use_cases.md) - Všechny Use Cases

**Rozhodnutí:**
- [decisions/uc1_workflow.md](../decisions/uc1_workflow.md) - Detailní UC-1
- [decisions/privacy_first_design.md](../decisions/privacy_first_design.md) - Privacy design (když bude vytvořen)

**Implementace:**
- [tasks/uc1_implementation.md](../tasks/uc1_implementation.md) - UC-1 status

**Technická dokumentace:**
- [base/database_schema.md](./database_schema.md) - DB tabulky
- [base/pipeline_recognition.md](./pipeline_recognition.md) - Recognition workflow

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Status:** ✅ ACTIVE - Reference pro implementaci

---
