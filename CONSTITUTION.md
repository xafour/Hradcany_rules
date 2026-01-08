# 🏛️ ÚSTAVA PROJEKTU HRADČANY

**Verze:** 0.9.2  
**Datum:** 2026-01-08  
**Status:** ✅ DRAFT (ACTIVE)

---

## 🎯 ÚČEL

Tento dokument definuje základní pravidla pro zachování kontinuity projektu. Tato pravidla jsou považována při práci s projektem jako neporušitelná.

## 📜 ZÁKLADNÍ PRAVIDLA

### **PRAVIDLO #1: Základní dokumenty a rozhodnutí se nikdy nemažou**

- Základní dokumenty popisují architekturu projektu a jsou uloženy ve složce `base/`.
- Každé důležité rozhodnutí zapíšeme do samostatného souboru ve složce `decisions/`.
- Tyto dokumenty a rozhodnutí jsou měnitelné pouze novou verzí. Jednou napsané se už nikdy nemažou.
- Číslo verze a datum aktualizace je uedeno v dokumentu, soubor s novou verzí má stejné jméno jako stará verze.

**Důvod:** Jednou rozhodnuté se v průběhu času neztratí a souvislosti při další práci na projektu nejsou v čase zapomenuty. 

**Kontrola:** Na konci každého chatu zkontrolovat jestli v `base/` a v `decisions/` nebyly smazány soubory.

---

### **PRAVIDLO #2: Seznam základních dokumentů a rozhodnutí**

- Existuje mapa dokumentace v souboru `INDEX.md` se seznamem všech základních dokumentů a rozhodnutí. V tomto souboru je uvedeno, kterou část dokumentace číst a kdy.  
- Kapitola **TIER 1** obsahuje dokumenty, které se čtou vždy, v každém chatu.  
- Kapitola **TIER 2** obsahuje další dokumenty rozdělené do kapitol, které se čtou selektivně, podle aktuálně řešeného úkolu. 
- Každý dokument v `base/` a `decisions/` je uveden v `INDEX.md`.
---

### **PRAVIDLO #3: Každý nový chat začíná čtením dokumentace**

- První prompt pro nový chat obsahuje informaci, co je dalším úkolem, jakou částí projektu se budeme v daném chatu zaobírat. 
- Nový chat začínáme čtením celého textu všech základních dokumentů **TIER 1** a seznamem všech dokumentů a zaznamenaných rozhodnutí **TIER 2**.
- Z iniciace promtu vybereme kapitolu z **TIER 2**, která bude přečtená celá a přečteme ji. Její přečtení bude explicitně potvrzeno.

Teprve potom může začít pracovat.

**Důvod:** 
- AI si obnoví kontext projektu, nemusí se ptát na věci, které jsme už vyřešili a bude postupovat s povědomím o detailech projektu.  
- Bude vědět, co již bylo naprogramováno a jak a nebude vytvářet duplicitní zdrojový kód.

**Kontrola:** Vzájmené potvrzení přečtení".

---

### **PRAVIDLO #4: Konec každého chatu musí projít kontrolou**

- Na konci každého chatu, kdy jsme něco změnili v dokumentaci, musí člověk zkontrolovat co AI změnilo a schválit to.
- Pokud kontrola nemůže proběhnout (třeba kvůli token limit), musí proběhnout obnovení dle recovery scénářů (viz RECOVERY.md).

AI napíše: "Prosím zkontroluj změny v dokumentaci pomocí: git diff". Člověk si zobrazí změny, zkontroluje jestli AI nepřepsalo základní dokumenty nebo needitovalo staré rozhodnutí. Teprve po schválení se změny commitnou.

**Důvod:** Hlavní ochrana proti tomu, aby se dokumenty postupně zkracovaly a ztrácely důležité informace.

**Kontrola:** Jednoduchá - na konci chatu `git diff` a schválení nebo zamítnutí změn.

---

### **PRAVIDLO #5: Když zjistíme ztrátu kontextu, máme postup jak ho obnovit**

- Když v průběhu chatu zjistíme, že AI něco zapomnělo nebo se ptá na věci které jsme už vyřešili, zastavíme všechnu práci. 
- V historii dokumentace najdeme kdy a jak jsme to rozhodli. Pomocí `git log`nebo procházením  najdeme v historii.
- Pokud chybí kontext, který byl zadán pouze do některého promptu v minulosti a v dalším průběhu projektu se ztratil z aktivních znalostí (to je nejčastější situace), bude popsán formou nového rozhodnutí.
- Přečteme si rozhodnutí a pak pokračujeme.

**Důvod:** Kontext arozhodnutí, které popíšeme formou, která se neztatí v promptech, se nemusí opakovaně popisovat v jednotlivých chatech. 

**Kontrola:** Když člověk řekne "stop, tohle jsme už řešili", přestaneme, vytvoříme dokument s informací a pak pokračujeme.

---

**Tento dokument je sám immutable (verze v0.9.2).**  
**Změna = nová verze (v1).**