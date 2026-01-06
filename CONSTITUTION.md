# 🏛️ ÚSTAVA PROJEKTU HRADČANY

**Verze:** 0.9.1  
**Datum:** 2026-01-06  
**Status:** ✅ DRAFT (ACTIVE)

---

## 🎯 ÚČEL

Tento dokument definuje základní pravidla pro zachování kontinuity projektu. Tato pravidla jsou považována při práci s projektem jako neporušitelná.

## 📜 ZÁKLADNÍ PRAVIDLA

### **PRAVIDLO #1: Základní dokumenty a rozhodnutí se nikdy nemažou**

Základní dokumenty popisují architekturu projektu a jsou uloženy ve složce `base/`.

Každé důležité rozhodnutí zapíšeme do samostatného souboru ve složce `decisions/`.

Tyto dokumenty a rozhodnutí jsou měnitelné pouze novou verzí. Jednou napsané se už nikdy nemažou.

Číslo verze a datum aktualizace je uedeno v dokumentu, soubor s novou verzí má stejné jméno jako stará verze.

**Důvod:** Jednou rozhodnuté se v průběhu času neztratí a souvislosti při další práci na projektu nejsou v čase zapomenuty. 

**Kontrola:** Na konci každého chatu zkontrolovat jestli v `base/` a v `decisions/` nebyly smazány soubory.

---

### **PRAVIDLO #2: Seznam základních dokumentů a rozhodnutí**

Existuje mapa dokumentace v souboru `INDEX.md` se seznamem všech základních dokumentů a rozhodnutí. V tomto souboru je uvedeno, kterou část dokumentace číst a kdy.

---

### **PRAVIDLO #3: Každý nový chat začíná čtením dokumentace**

Když začínáme nový chat, začínáme čtením všech základních dokumentů a seznamem všech zaznamenaných rozhodnutí z minulosti.

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