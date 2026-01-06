# Hradčany Rules - Projektová dokumentace

**Verze:** 0.9  
**Datum:** 2026-01-06  
**Status:** ✅ DRAFT (ACTIVE)

Tento repozitář obsahuje dokumentaci designu, rozhodnutí a specifikací pro projekt rozpoznávání československých poštovních známek série Hradčany.

---

## 🎯 Účel

Tento repozitář slouží jako **znalostní báze** pro projekt Hradčany. Zajišťuje kontinuitu napříč desítkami nebo stovkami vývojových sessions a zabraňuje ztrátě kontextu.

---

## 📚 Struktura

Repozitář obsahuje následující soubory a složky:

- **`CONSTITUTION.md`** - Základní neporušitelná pravidla projektu (neměnnost, append-only, recovery postupy)
- **`INDEX.md`** - Mapa dokumentace (co číst v jaké situaci)
- **`PRINCIPLES.md`** - Kódovací principy a best practices
- **`RECOVERY.md`** - Recovery scénáře při ztrátě kontextu
- **`base/`** - Základní dokumentace (verzovaná, immutable)
- **`decisions/`** - Log designových rozhodnutí (append-only, datované soubory)

---

## 🏛️ Základní principy

Projekt je postaven na pěti základních principech:

1. **Neměnnost základních dokumentů** - Dokumenty v `base/` se nikdy nemažou, pouze se vytváří nové verze
2. **Append-only rozhodnutí** - Historie všech rozhodnutí zachovaná v `decisions/`, nikdy nesmazaná
3. **Povinné čtení na začátku** - Každá session začíná přečtením aktuální dokumentace
4. **Verifikace změn** - Všechny změny v dokumentaci kontrolovány člověkem před committem
5. **Recovery postupy** - Jasné postupy jak obnovit ztracený kontext

---

## 📖 Pro AI asistenty

Pokud jste AI asistent pracující na tomto projektu:

1. Přečtěte nejdřív `CONSTITUTION.md` (základní pravidla projektu)
2. Přečtěte `INDEX.md` abyste věděli co číst v dnešní situaci
3. Načtěte relevantní základní dokumenty z `base/` podle typu úkolu
4. Přečtěte nedávná rozhodnutí z `decisions/` (minimálně posledních 5)
5. Dodržujte pravidla striktně - jsou zde proto, aby se zabránilo ztrátě kontextu

---

## 🔄 Status projektu

**Verze dokumentace:** 0.9 (DRAFT)  
**Poslední update:** 2026-01-06  
**Aktivní vývoj:** Ano

---

## 🎓 O projektu

Projekt Hradčany se zabývá automatickým rozpoznáváním československých poštovních známek série Hradčany z roku 1918 pomocí computer vision technologií. Cílem je identifikovat nejen nominál známky, ale i konkrétní tiskovou desku (TD) a známkové pole (ZP) na základě drobných rozdílů v tisku.

Dokumentace v tomto repozitáři popisuje workflow, designová rozhodnutí a architektonické principy projektu.

---