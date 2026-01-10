# 📚 INDEX DOKUMENTACE - Projekt Hradčany

**Verze:** 2.0.0  
**Datum:** 2026-01-10  
**Účel:** Mapa dokumentace - kterou část číst a kdy

---

## 🎯 TIER 1: ČÍST VŽDY (každý nový chat)

Tyto dokumenty se čtou VŽDY na začátku každého nového chatu:

1. **CONSTITUTION.md** - Základní pravidla projektu
2. **PRINCIPLES.md** - Principy a mantry
3. **PROJECT_STATUS.md** - Aktuální status projektu
4. **RECOVERY.md** - Recovery scénáře při ztrátě kontextu
5. **ARCHITECTURE.md** - High-level architektura systému

---

## 📖 TIER 2: ČÍST SELEKTIVNĚ (podle úkolu)

Tyto dokumenty se čtou podle aktuálně řešeného úkolu.

### 🔵 KAPITOLA 1: Database Structure

**Čti když:** Potřebuješ znát strukturu tabulek, sloupce, vztahy  
**Přeskoč když:** Pracuješ jen s high-level workflow bez DB interakce

- `base/database_schema.md` - DB schema v3.1.0 (1:1 link na DB_STRUKTURA_PRUHLEDCE.md)

---

### 🔵 KAPITOLA 2: Recognition Pipeline

**Čti když:** Implementuješ cokoliv co pracuje se skeny, potřebuješ správné file handling  
**Přeskoč když:** Pracuješ jen s UI nebo databází bez file I/O

- `base/pipeline_recognition.md` ⭐ **ZLATÝ STANDARD** - Jak SPRÁVNĚ ukládat soubory (Fáze 2 a 8)

---

### 🔵 KAPITOLA 3: GT Management System

**Čti když:** Začínáš práci na GT Management nebo implementuješ jakýkoliv UC  
**Přeskoč když:** Pracuješ jen na recognition pipeline

- `decisions/gt_management_use_cases.md` ⭐ - Celkový koncept (17 UC, 4 workflows)
- `decisions/uc1_workflow.md` - UC-1 Upload & Analyze (11 kroků detailně)
- `base/gt_workflows.md` - WF-1 až WF-4 implementační detaily

---

### 🔵 KAPITOLA 4: File Storage

**Čti když:** Implementuješ file upload/download, debugging storage issues  
**Přeskoč když:** Pracuješ jen s embeddingy nebo DB

- `base/file_storage_architecture.md` - SHA256 storage, path reconstruction
- `decisions/sha256_storage_strategy.md` - Proč SHA256, deduplication, multi-user

---

### 🔵 KAPITOLA 5: Embeddings & Models

**Čti když:** Pracuješ s embeddingy nebo RealEncoder  
**Přeskoč když:** Pracuješ jen s file storage nebo DB

- `decisions/realencoder_centralization.md` - Proč centralizace, RealEncoder vs ResNet50Embed

---

### 🔵 KAPITOLA 6: Database Design Decisions

**Čti když:** Potřebuješ pochopit proč DB schema je navržené takhle  
**Přeskoč když:** Jen používáš DB bez změn schema

- `decisions/schema_v3_1_0_nullable_columns.md` - Proč nullable columns, pending workflow

---
