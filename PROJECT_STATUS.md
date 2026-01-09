# 🏗️ PROJECT STATUS - Projekt Hradčany

**Poslední aktualizace:** 2025-12-31  
**Aktuální verze:** 3.1.0 (GT Management - UC-1 Complete)  
**Status:** ✅ PRODUCTION-READY RECOGNITION + UC-1 UPLOAD WORKFLOW COMPLETE

---

## 🎯 CURRENT MILESTONE: GT Management Implementation

**Phase:** GT Management (Multi-user System)  
**Status:** 🟢 UC-1 COMPLETE → UC-2 NEXT  
**Priority:** User confirmation workflow (UC-2)

---

## 📊 SYSTEM PERFORMANCE

### Recognition Pipeline (v3.0.0)
- **Overall:** 96.9% success rate (6,800 baseline scans)
- **8 denominations:** 100% accuracy
- **10 denominations:** 99%+ accuracy
- **Multi-phase:** Embedding → OCR → HSV validation

### GT Upload Workflow (v1.6.0) ✨ NEW!
- **UC-1:** Upload & Analyze - ✅ COMPLETE
- **Steps 1-11:** Full workflow operational
- **Auto-detect:** HIGH confidence (embedding + OCR + HSV)
- **Duplicate detection:** SHA256-based
- **Multi-user:** Ownership tracking implemented

---

## ✅ COMPLETED FEATURES

### 🎯 TIER 1: Core Recognition (COMPLETE)
- ✅ YOLO frame detection (OBB)
- ✅ ResNet50 embeddings with deterministic projection head
- ✅ Multi-phase denomination detection (Embedding → OCR → HSV)
- ✅ OCR integration (EasyOCR)
- ✅ HSV color validation
- ✅ Confidence levels (HIGH/MEDIUM/LOW/MANUAL_REVIEW)
- ✅ Batch testing infrastructure (6,800 baseline scans)

### 🗄️ TIER 2: Database Architecture (v3.1.0)
- ✅ Schema v3.0.0: reference_scans → reference_front
- ✅ Schema v3.1.0: Nullable metadata columns ✨ NEW!
- ✅ Multi-user tables: user_uploads, scan_supplementary_files
- ✅ Quality review: suspect_flag, suspect_reason
- ✅ Audit logging: gt_audit_log
- ✅ 6,800 baseline scans + 33,000 embeddings

### 🔧 TIER 3: Utilities & Infrastructure (COMPLETE)
- ✅ db_utils.py: Centralized DB operations (22 functions)
- ✅ embedding_utils.py v1.3.0: RealEncoder class ✨ NEW!
- ✅ denomination_utils.py v2.0.1: Multi-phase auto-detect
- ✅ ocr_utils.py: EasyOCR wrapper
- ✅ color_utils.py: HSV matching
- ✅ yolo_utils.py: Detection utilities
- ✅ paths.json v2.1: Centralized path management

### 🎯 GT MANAGEMENT: UC-1 Upload & Analyze (COMPLETE) ✨ NEW!
- ✅ **Step 1:** YOLO detection (conf > 0.95)
- ✅ **Step 2:** Warp to 1300x1100
- ✅ **Step 3:** SHA256 computation
- ✅ **Step 4:** Duplicate detection
- ✅ **Step 4b:** Duplicate re-analysis (LIVE)
- ✅ **Step 5:** SHA-based file storage (256 directories)
- ✅ **Step 6:** DB insert (reference_front, confirmed=0)
- ✅ **Step 7:** User ownership tracking (user_uploads)
- ✅ **Step 8:** YOLO cache (inference_frames)
- ✅ **Step 9:** Supplementary files (back/cert/block)
- ✅ **Step 10:** Auto-detect denomination (HIGH confidence)
- ✅ **Step 11:** Audit log (action='added_to_gt')

**Module:** `common/gt_upload_utils.py` v1.6.0  
**Features:** Debug mode, duplicate handling, multi-file support

---

## 🚧 IN PROGRESS

### GT MANAGEMENT: UC-2 Confirm Classification (NEXT)
- ⏳ Expert/user confirms auto-detected classification
- ⏳ Update metadata (stamp_type_id, plate_id, zp_no)
- ⏳ Set confirmed=1 (GT verified)
- ⏳ Trigger embedding computation
- ⏳ Remove from pending queue

**Estimated:** 2-3 hours implementation

---

## 📋 TODO (Prioritized)

### Phase 2A: GT Management (User Workflows)
1. **UC-2:** Confirm Classification ⏳ NEXT (2h)
2. **UC-3:** View My Uploads (1h)
3. **UC-4:** Delete/Edit Upload (1h)

### Phase 2B: GT Management (Expert Workflows)
4. **UC-5:** Expert Review Queue (2h)
5. **UC-6:** Approve/Reject Submissions (2h)
6. **UC-7:** Batch Approval (1h)
7. **UC-8:** Quality Review (suspect flags) (1h)

### Phase 3: Production Deployment
8. **Prod separation:** dev/ → prod/ migration
9. **Web UI:** Flask/FastAPI integration
10. **Authentication:** Multi-user access control
11. **API endpoints:** REST API for uploads

### Phase 4: Advanced Features
12. **Mass import:** Auction scraping (expand to 10,000+ scans)
13. **Periodic review:** Auto-flagging for quality check
14. **Advanced search:** Filter by denomination, TD, ZP
15. **Statistics dashboard:** Upload trends, detection accuracy

---

## 🔬 KNOWN ISSUES

### High Priority (Blocking UC-2)
- None! UC-1 fully operational ✅

### Medium Priority (Non-blocking)
1. **1000h OCR:** 58% accuracy (OCR reads "100"/"10" instead of "1000")
   - Workaround: Trust embedding similarity
   - Fix: Better OCR preprocessing, larger crop
2. **15h TD4:** ~5% accuracy (quality issues with TD4 scans)
   - Workaround: Separate handling for TD4
   - Fix: Review TD4 baseline quality

### Low Priority (Future)
3. **paths.json shared_dir:** Returns `/dev/data` instead of `/shared`
   - Impact: Model path resolution needs fallback
   - Fix: Correct paths.json or load_config.py logic

---

## 📦 MODULE VERSIONS

### Core Modules
- `recognize_stamp.py`: v3.0.0 (production recognition)
- `gt_upload_utils.py`: v1.6.0 ✨ (UC-1 complete)
- `embedding_utils.py`: v1.3.0 ✨ (RealEncoder)
- `denomination_utils.py`: v2.0.1 (multi-phase)
- `db_utils.py`: v1.0.0 (22 functions)
- `load_config.py`: v1.9.1

### Database
- Schema version: v3.1.0 ✨
- Baseline scans: 6,800
- Embeddings: 33,000
- User uploads: 1 (test)

### Models
- YOLO: `best.pt` (ramec-obb_640)
- Embedding: ResNet50 + deterministic head (seed=12345)
- Model ID: 1

---

## 🎓 KEY LEARNINGS

### Session 2025-12-30/31 ✨
1. **RealEncoder vs ResNet50Embed:**
   - RealEncoder = wrapper with preprocessing + deterministic head
   - ResNet50Embed = raw model (no preprocessing in __call__)
   - ALWAYS use RealEncoder in production!

2. **auto_detect_denomination() returns:**
   ```python
   {
     'denomination': '500h',
     'confidence_level': 'HIGH',
     'stamp_type_id': 5,
     'similarity': 0.851,
     ...
   }
   ```
   NOT `{'matches': [...]}` - important for result parsing!

3. **SQLite ALTER limitations:**
   - No `ALTER COLUMN DROP NOT NULL`
   - Must use rebuild strategy (create new, copy, drop old, rename)
   - Always backup before schema migrations!

4. **Debug mode pattern:**
   - Add `debug: bool = False` parameter
   - Helper function: `def debug_print(msg): if debug: print(msg)`
   - Production-ready by default, verbose on demand

### Historical Learnings
1. **Multi-phase detection > single method** (embedding + OCR + HSV)
2. **Confidence levels matter** (HIGH/MEDIUM/LOW/MANUAL_REVIEW)
3. **Batch testing validates production** (6,800 baseline sanity check)
4. **Root cause > workaround** (fix indentation, not shimming)
5. **Single source of truth** (RealEncoder in embedding_utils, not duplicated)

---

## 🔄 VERSION HISTORY

### v3.1.0 (2025-12-31) - Schema Migration + UC-1 Complete
- ✅ Schema: Nullable metadata columns (stamp_type_id, plate_id, zp_no)
- ✅ gt_upload_utils.py v1.6.0: UC-1 all 11 steps working
- ✅ RealEncoder centralized in embedding_utils.py v1.3.0
- ✅ Debug mode implemented (conditional logging)
- ✅ Audit log working (action='added_to_gt')
- ✅ Duplicate path tested and verified

### v3.0.0 (2025-12-20) - GT Management Design
- DB rename: reference_scans → reference_front
- New tables: user_uploads, scan_supplementary_files
- Quality review: suspect_flag, suspect_reason
- 17 use cases defined across 4 workflows

### v2.4.0 (2025-12-16) - OCR Bug Fixes
- Fixed denomination s indexem (10h5, 20h9)
- Extended TOP-10 → TOP-20 OCR matching
- 96.9% success rate achieved

### v2.0.0 (2025-12-13) - Auto-Detect Denomination
- Multi-phase pipeline: Embedding → OCR → HSV
- Confidence levels implementation
- EasyOCR integration

---

## 📞 CONTACTS & RESOURCES

- **Project Owner:** Milan (@zenbook)
- **Repository:** GitHub (local development)
- **Environment:** Ubuntu 24, Python 3.12, PyTorch
- **Hardware:** Zenbook (CPU dev), Desktop (GPU - future)

---

**🎉 MILESTONE REACHED: UC-1 Upload & Analyze Complete!**  
**🎯 NEXT: UC-2 Confirm Classification**

---
*Last updated: 2025-12-31 by Claude (Session: GT Upload Implementation)*
