# Hradčany Rules - Projektová dokumentace

**Verze:** 0.9.8  
**Datum:** 2026-01-09  

Tento repozitář obsahuje dokumentaci designu, rozhodnutí a specifikací pro projekt rozpoznávání československých poštovních známek série Hradčany.

---

## 🎯 Účel

Tento repozitář slouží jako **znalostní báze** pro projekt Hradčany. Zajišťuje kontinuitu napříč desítkami nebo stovkami vývojových sessions a zabraňuje ztrátě kontextu.

---

## 📚 Struktura

Repozitář obsahuje následující soubory a složky:

- **[CONSTITUTION.md](./CONSTITUTION.md)** - Základní neporušitelná pravidla projektu
- **[INDEX.md](./INDEX.md)** - Mapa dokumentace (co číst v jaké situaci)
- **[PRINCIPLES.md](./PRINCIPLES.md)`.md`** - Kódovací principy a best practices
- **[RECOVERY.md](./RECOVERY.md)`.md`** - Recovery scénáře při ztrátě kontextu
- **[base/](./base/)** - Základní dokumentace
- **[decisions/](./decisions/)** - Log rozhodnutí

---

## 🎓 O projektu

Projekt Hradčany se zabývá automatickým rozpoznáváním československých poštovních známek série Hradčany z roku 1918 pomocí computer vision technologií. Cílem je identifikovat nejen nominál známky, ale i konkrétní tiskovou desku (TD) a známkové pole (ZP) na základě drobných rozdílů v tisku.

Dokumentace v tomto repozitáři popisuje workflow, designová rozhodnutí a architektonické principy projektu.

---