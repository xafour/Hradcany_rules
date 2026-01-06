# 📚 INDEX - Mapa dokumentace

Tento soubor říká co číst kdy. Ne všechno je potřeba vždy.

**Verze:** 0.9  
**Datum:** 2026-01-06  
**Status:** ✅ DRAFT (ACTIVE)

---

## 🎯 TIER 1: VŽDY (každý chat)

**Mandatory reading před začátkem práce:**

1. **CONSTITUTION.md** - Pravidla projektu (5 zákonů)
2. **INDEX.md** - Tento soubor (co číst dnes)
3. **Posledních 5 decisions/** - Nedávná rozhodnutí

---

## 📖 TIER 2: PODLE ÚKOLU (selektivní)

### **Architektura & Pipeline**

**ARCHITECTURE.md**
- **Čti když:** Změny v pipeline, refaktoring, nové moduly
- **Přeskoč když:** UI změny, dokumentace updates
- **Obsah:** Pipeline overview, moduly, DB schema, file storage

**RECOGNIZE_STAMP_FLOW.md**
- **Čti když:** Práce na recognize_stamp.py, auto-detect, embedding matching
- **Přeskoč když:** DB changes, GT workflow, UI
- **Obsah:** Krok-po-kroku pipeline, auto-detect logika

---

### **Domain Knowledge**

**DOMAIN_KNOWLEDGE.md**
- **Čti když:** První chat, nepochopení filatelistického kontextu
- **Přeskoč když:** Refaktoring známého kódu
- **Obsah:** Co je TD, ZP, kresba, historický kontext známek

---

### **GT Management**

**GT_WORKFLOW.md**
- **Čti když:** Práce na UC-1 až UC-17
- **Přeskoč když:** Recognition pipeline, model training
- **Obsah:** User workflows, expert workflows, use cases

---

## 🔍 JAK POUŽÍT TENTO INDEX

### **Příklad 1: Task "Refaktorovat ukládání souborů"**

**MANDATORY:**
- ✅ CONSTITUTION.md
- ✅ INDEX.md
- ✅ Posledních 5 decisions/

**SELEKTIVNÍ:**
- ✅ ARCHITECTURE.md (sekce File Storage)
- ⚠️ DOMAIN_KNOWLEDGE.md (rychlé pročtení pro kontext)
- ❌ RECOGNIZE_STAMP_FLOW.md (nepotřeba)
- ❌ GT_WORKFLOW.md (nepotřeba)

**DECISIONS:**
- ✅ Všechny obsahující "file", "storage", "SHA"

---

### **Příklad 2: Task "Implementovat UC-2 Confirm Classification"**

**MANDATORY:**
- ✅ CONSTITUTION.md
- ✅ INDEX.md
- ✅ Posledních 5 decisions/

**SELEKTIVNÍ:**
- ✅ GT_WORKFLOW.md (kompletně!)
- ✅ ARCHITECTURE.md (sekce GT Management)
- ⚠️ RECOGNIZE_STAMP_FLOW.md (souvislost s auto-detect)
- ❌ DOMAIN_KNOWLEDGE.md (už známo)

---

## 📌 POZNÁMKY

- **Když si nejsi jistý:** Radši přečti více než méně
- **První chat projektu:** Přečti VŠECHNO v base/
- **Ztráta kontextu:** Přečti VŠECHNO znovu