# 🏛️ ÚSTAVA PROJEKTU HRADČANY

**Verze:** 0.9  
**Datum:** 2026-01-06  
**Status:** ✅ DRAFT (ACTIVE)

---

## 🎯 ÚČEL

Tento dokument definuje neporušitelná pravidla pro kontinuitu projektu přes desítky nebo stovky chatů s AI asistenty a možné změny v AI systémech.

---

## 📜 ZÁKLADNÍ PRAVIDLA

### **PRAVIDLO #1: Základní dokumenty se nikdy nepřepisují**

Ve složce `base/` jsou uloženy základní dokumenty popisující architekturu projektu. Tyto dokumenty jsou neměnné - jednou napsané se už nikdy nepřepisují ani neupravují.

Když potřebujeme změnit něco v základním dokumentu, nevypouštíme starou verzi. Místo toho vytvoříme nový soubor s vyšším číslem verze. Například pokud máme `ARCHITECTURE_v1.md` a potřebujeme ho aktualizovat, vytvoříme `ARCHITECTURE_v2.md`. Starý soubor `v1` smažeme, ale zůstane v Git historii.

**Důvod:** Díky tomu máme vždy k dispozici původní informace v Git historii. Když zjistíme, že jsme něco zapomněli, můžeme se kdykoliv vrátit k předchozí verzi.

**Kontrola:** Na konci každého chatu zkontrolovat změny v `base/` pomocí `git diff base/`.

---

### **PRAVIDLO #2: Rozhodnutí se jen přidávají, nikdy nemazat**

Každé důležité rozhodnutí zapíšeme do samostatného souboru ve složce `decisions/`. Tyto soubory už nikdy nemažeme - pouze přidáváme nové, nebo vytvoříme novou verzi stávajícího rozhodnutí.

Když v nějakém chatu rozhodneme něco důležitého (například "TOP-5 kandidátů se neukládají do databáze"), vytvoříme nový soubor s datem a názvem rozhodnutí, například `2025-12-31_top_k_no_db.md`. Pokud později změníme rozhodnutí, vytvoříme novou verzi tohoto dokumentu s novým datem.

**Důvod:** Máme tak kompletní historii všech rozhodnutí včetně vysvětlení proč jsme to rozhodli.

**Kontrola:** Na konci každého chatu zkontrolovat změny v `decisions/` - povoleno pouze přidávání nových souborů nebo nové verze, žádný dokument nesmí být smazán.

---

### **PRAVIDLO #3: Každý nový chat začíná čtením dokumentace**

Když začínáme nový chat, AI asistent musí nejdříve přečíst všechny základní dokumenty a všechna zaznamenaná rozhodnutí z minulosti.

Na začátku každého nového chatu AI napíše: "Čtu dokumentaci před začátkem práce..." a postupně načte `CONSTITUTION.md`, `INDEX.md` a další soubory v hlavním adresáři, relevantní soubory z `base/` a nedávná rozhodnutí z `decisions/`. Teprve potom může začít pracovat.

**Důvod:** 
- AI si obnoví kontext projektu a nemusí se ptát na věci, které jsme už vyřešili.  
- Bude vědět, co již bylo naprogramováno a jak a nebude vytvářet duplicitní zdrojový kód.

**Kontrola:** Pokud AI na začátku chatu rovnou začne programovat, zastavit ho: "Stop, nejdřív si přečti dokumentaci".

---

### **PRAVIDLO #4: Konec každého chatu musí projít kontrolou**

Na konci každého chatu, kdy jsme něco změnili v dokumentaci, musí člověk zkontrolovat co AI změnilo a schválit to.

Pokud kontrola nemůže proběhnout (třeba kvůli token limit), musí proběhnout obnovení dle recovery scénářů (viz RECOVERY.md).

AI napíše: "Prosím zkontroluj změny v dokumentaci pomocí: git diff". Člověk si zobrazí změny, zkontroluje jestli AI nepřepsalo základní dokumenty nebo needitovalo staré rozhodnutí. Teprve po schválení se změny commitnou.

**Důvod:** Hlavní ochrana proti tomu, aby se dokumenty postupně zkracovaly a ztrácely důležité informace.

**Kontrola:** Jednoduchá - na konci chatu `git diff` a schválení nebo zamítnutí změn.

---

### **PRAVIDLO #5: Když zjistíme ztrátu kontextu, máme postup jak ho obnovit**

Když v průběhu chatu zjistíme, že AI něco zapomnělo nebo se ptá na věci které jsme už vyřešili, zastavíme všechnu práci. AI řekne: "Hledám v historii..." a pomocí `git log` najde kdy a jak jsme to rozhodli. Přečte si původní rozhodnutí a pak pokračujeme.

**Důvod:** Chyby se stanou. Důležité není chyby nedělat, ale mít jasný způsob jak se z nich dostat. Díky Git historii a decision souborům můžeme vždy najít co jsme ztratili.

**Kontrola:** Když člověk řekne "stop, tohle jsme už řešili", AI okamžitě přestane, najde v historii správnou informaci a pak pokračujeme.

---

## ✅ END-OF-CHAT CHECKLIST

**Pro člověka:**
- [ ] `git diff` zkontrolován
- [ ] Žádné změny v `base/` (nebo jen nová verze)
- [ ] Žádné edity starých `decisions/`
- [ ] Nové decisions reviewed
- [ ] Commit approved

---

**Tento dokument je sám immutable (verze v0.9).**  
**Změna = nová verze (v2).**