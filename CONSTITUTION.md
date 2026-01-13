# 🏛️ ÚSTAVA PROJEKTU HRADČANY

**Verze:** 1.0.1  
**Datum:** 2026-01-13  

---

## 🎯 ÚČEL

Tento dokument definuje základní pravidla pro zachování kontinuity projektu. Tato pravidla jsou považována při práci s projektem jako neporušitelná.

## 📜 ZÁKLADNÍ PRAVIDLA

### **PRAVIDLO #1: Základní dokumenty a rozhodnutí se nikdy nemažou**

#### **base/** - Odborná a technická dokumentace systému a problematiky
- Obsahuje základní dokumenty popisují architekturu projektu a jsou uloženy ve složce `base/`.

  ***CO SEM PATŘÍ:***
  - Popis souvislostí odborné problematiky známek série Hradčany se systémem
  - Popis architektury (pipeline, database schema, file storage)
  - Workflow diagramy (jak systém funguje)
  - API reference (jaké funkce existují)
  - Best practices (jak správně používat systém)

#### **decisions/** - Rozhodnutí a jejich odůvodnění
- Každé důležité rozhodnutí zapíšeme do samostatného souboru ve složce `decisions/`.

  ***CO SEM PATŘÍ:***
  - Popis rozhodnutí (CO jsme se rozhodli)
  - Odůvodnění (PROČ jsme se tak rozhodli)
  - Souvislosti (jaké faktory jsme zohlednili)
  - Důsledky (co z toho vyplývá pro implementaci)

#### **tasks/** - Implementační kroky a milestones
- Obsahuje dokumentaci implementačních kroků

  ***CO SEM PATŘÍ:***
  - Plánované tasky
  - Dokončené milestones
  - Status tracking (co je uděláno, jaké jsou další implementační kroky, priority)
  - Handover messages mezi chaty (kde jsme skončili, co pokračuje)

#### Pravidla práce s dokumenty
- Tyto dokumenty a rozhodnutí jsou měnitelné pouze novou verzí. Jednou napsané se už nikdy nemažou.  
- Číslo verze a datum aktualizace je uvedeno v dokumentu, soubor s novou verzí má stejné jméno jako stará verze.  

**Důvod:** Jednou rozhodnuté se v průběhu času nezapomene a souvislosti při další práci na projektu nejsou v čase zapomenuty. 

**Kontrola:** Na konci každého chatu zkontrolovat jestli v `base/` a v `decisions/` nebyly smazány soubory.

---

### **PRAVIDLO #2: Seznam základních dokumentů a rozhodnutí**

- Existuje mapa dokumentace v souboru `INDEX.md` se seznamem všech základních dokumentů a rozhodnutí. V tomto souboru je uvedeno, kterou část dokumentace číst a kdy.  
- Kapitola **TIER 1** obsahuje dokumenty, které se čtou vždy, v každém chatu.  
- Kapitola **TIER 2** obsahuje další dokumenty rozdělené do kapitol, které se čtou selektivně, podle aktuálně řešeného úkolu. 
- Každý dokument v `base/` a `decisions/` je uveden v `INDEX.md`.
---

### **PRAVIDLO #3: Každý nový chat začíná čtením dokumentace**

- První prompt pro nový chat obsahuje informaci, jaký je úkol pro tento chat.
- Nový chat začínáme čtením celého textu všech základních dokumentů **TIER 1** a seznamem všech dokumentů a zaznamenaných rozhodnutí **TIER 2**.
- Z iniciace promptu vybereme kapitolu z **TIER 2**, která bude přečtená celá a přečteme ji. Její přečtení bude explicitně potvrzeno.

Teprve potom může začít pracovat.

**Důvod:** 
- AI si obnoví kontext projektu, nemusí se ptát na věci, které jsme už vyřešili a bude postupovat s povědomím o detailech projektu.  
- Bude vědět, co již bylo naprogramováno a jak a nebude vytvářet duplicitní zdrojový kód.

**Kontrola:** Vzájemné potvrzení přečtení.

---

### **PRAVIDLO #4: Na konci každého chatu budou zkušenosti a rozhodnutí zaznamenány**

- Na konci každého chatu budou zkušenosti z aktuálního chatu zaznamenány do souborů dokumentace v `base/` a `decisions/`.
- Každý dokument MUSÍ být psán jako popisný text v celých větách (podmět + přísudek), jako firemní směrnice nebo rozhodnutí vedení. Odrážky s hesly mohou být použity pouze jako doplňková struktura k již vysvětlenému textu.
- Když jsme něco změnili v dokumentaci, musí člověk změny zkontrolovat a schválit je.
- Pokud zaznamenání nemůže proběhnout (třeba kvůli token limit), musí proběhnout obnovení dle recovery scénářů (viz RECOVERY.md).

Člověk si zobrazí změny, zkontroluje jestli nejsou přepsány základní dokumenty nebo editována stará rozhodnutí bez změny verze. Teprve po schválení se změny commitnou.

**Důvod:** Hlavní ochrana proti tomu, aby se dokumenty postupně zkracovaly a ztrácely důležité informace. Dokumenty v `base/` a `decisions/` slouží jako dlouhodobá firemní znalostní báze. Musí být srozumitelné i po měsících nebo letech, když se k nim vrátíme. Heslovitý styl s odrážkami vede k postupné erozi kontextu.

**TEMPLATES:**
Ve složce `base/` se pro:
- popis souvislostí odborné problematiky známek série Hradčany se systémem použije template `base_template_domain.md` v této složce.
- pro popis technické dokumentace se použije template `base_template_tech.md` v této složce.
Pro popis rozhodnutí v `decisions/` se použije template `decisions_template.md` v této složce.
Pro popis úkolů v `tasks/` se použije template `task_template.md` v této složce.

**ENFORCEMENT:**
- AI nikdy nevytvoří dokument v `base/` nebo `decisions/` bez popisného úvodního odstavce vysvětlujícího CO a PROČ
- Technické detaily (SQL, kód, příklady) jsou vždy AŽ PO vysvětlujícím textu
- Heslovité odrážky jsou povoleny pouze jako doplňková navigace
- Při code review (git diff) člověk zkontroluje, že nový/upravený dokument splňuje tento požadavek

**KONTROLA:**
Na konci chatu se člověk zeptá: "Rozumím tomuto dokumentu i když se k němu vrátím za rok?" Pokud ne → dokument je heslovitý → AI musí přepsat popisně.
---

### **PRAVIDLO #5: Když zjistíme ztrátu kontextu, máme postup jak ho obnovit**

- Když v průběhu chatu zjistíme, že AI něco zapomnělo nebo se ptá na věci které jsme už vyřešili, zastavíme všechnu práci. 
- V historii dokumentace najdeme kdy a jak jsme to rozhodli. Pomocí `git log`nebo procházením  najdeme v historii.
- Pokud chybí kontext, který byl zadán pouze do některého promptu v minulosti a v dalším průběhu projektu se ztratil  
  z aktivních znalostí (to je nejčastější situace), bude popsán formou nového rozhodnutí.
- Přečteme si rozhodnutí a pak pokračujeme.

**Důvod:** Kontext a rozhodnutí popsaná ve formě, která se neztratí v jednotlivých promptech, se nemusí opakovaně popisovat. 

**Kontrola:** Když člověk řekne "stop, tohle jsme už řešili", přestaneme, vytvoříme dokument s informací a pak pokračujeme.

---
