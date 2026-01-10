# [NÁZEV KOMPONENTY/SUBSYSTÉMU]

**Tags:** `#technická-dokumentace` `#tag2` `#tag3`  
**Verze:** X.Y.Z  
**Datum:** YYYY-MM-DD  
**Status:** Active / Draft / Deprecated

---

## 🎯 ÚČEL

[POPISNÝ TEXT - minimálně 2-3 odstavce celých vět!]

Tento dokument popisuje [název komponenty/subsystému]. Slouží jako referenční dokumentace pro pochopení toho, jak daná část systému funguje, jaké má odpovědnosti a jak s ní pracovat.

Vysvětli:
- CO tento subsystém/komponenta dělá?
- PROČ to dělá (jaký problém řeší)?
- JAK to zapadá do celkového systému?

---

## 📋 PŘEHLED

[POPISNÝ TEXT - celé věty!]

Stručné shrnutí hlavních konceptů a komponent, které budou v dokumentu popsány. Tento přehled pomáhá čtenáři rychle pochopit, o čem dokument je, aniž by musel číst všechny detaily.

Pokud je subsystém složitý, zde můžeme uvést high-level diagram nebo seznam hlavních částí s jednořádkovým popisem každé.

---

## 🏗️ ARCHITEKTURA / STRUKTURA

[POPISNÝ TEXT před jakýmikoli diagramy nebo odrážkami!]

Tato sekce popisuje, jak je daná komponenta/subsystém strukturována. Vysvětlujeme:
- Jaké hlavní části tvoří tento subsystém
- Jak spolu tyto části komunikují
- Jaké jsou hlavní datové toky
- Kde se co ukládá a proč

### [Podnázev - volitelný]

Pokud je potřeba rozdělit architekturu na více částí, každá část musí mít úvodní vysvětlující odstavec.

**Teprve po vysvětlení můžou následovat:**
- Diagramy
- Výpisy struktur
- Technické detaily
```
[diagram nebo technický detail]
```

---

## 🔄 WORKFLOW / JAK TO FUNGUJE

[POPISNÝ TEXT vysvětlující hlavní workflow!]

Tato sekce popisuje krok za krokem, co se děje při běžném použití této komponenty. Vysvětlujeme proces tak, jako bychom ho vysvětlovali kolegovi, který s ním zatím nepracoval.

### Krok 1: [Název kroku]

[Popisný text vysvětlující CO se děje v tomto kroku a PROČ]

Každý krok workflow musí být vysvětlen celými větami. Odrážky nebo číslované seznamy můžou následovat až PO vysvětlení jako doplněk.

**Technické detaily tohoto kroku:**
```python
# Ukázka kódu nebo SQL
```

### Krok 2: [Název kroku]

[Stejný formát jako krok 1]

---

## 🔑 KLÍČOVÉ KONCEPTY

[POPISNÝ TEXT!]

Tato sekce vysvětluje důležité koncepty, které je potřeba pochopit pro práci s touto komponentou. Každý koncept vysvětlujeme tak, aby byl srozumitelný i někomu, kdo s ním zatím nepracoval.

### Koncept 1: [Název]

[Popisný text - CO to je, PROČ to máme, JAK se to používá]

**Technické detaily:**
```python
# Ukázka použití
```

### Koncept 2: [Název]

[Stejný formát]

---

## ⚠️ DŮLEŽITÁ PRAVIDLA / BEST PRACTICES

[POPISNÝ TEXT!]

Tato sekce popisuje pravidla a osvědčené postupy, které MUSÍ být dodrženy při práci s touto komponentou. Každé pravidlo vysvětlujeme včetně důvodu - proč existuje a co se stane, když ho nedodržíme.

### Pravidlo 1: [Název pravidla]

[Popisný text vysvětlující CO je pravidlo, PROČ existuje, DŮSLEDKY při nedodržení]

**✅ Správně:**
```python
# Ukázka správného použití
```

**❌ Špatně:**
```python
# Ukázka chyby + komentář proč je to špatně
```

### Pravidlo 2: [Název]

[Stejný formát]

---

## 🔧 TECHNICKÁ REFERENCE

[POPISNÝ TEXT úvodem!]

Tato sekce obsahuje technické detaily pro pokročilé použití nebo implementaci. Předpokládáme, že čtenář již pochopil základní koncepty z předchozích sekcí.

### API / Rozhraní

[Popisný text vysvětlující účel API]
```python
def function_name(param1: Type, param2: Type) -> ReturnType:
    """
    Popisný docstring vysvětlující CO funkce dělá,
    PROČ existuje, JAK se používá.
    """
```

### Datové struktury

[Popisný text vysvětlující struktury]

### Konfigurace

[Popisný text o konfiguraci]

---

## 🐛 ZNÁMÉ PROBLÉMY / OMEZENÍ

[POPISNÝ TEXT!]

Tato sekce popisuje známá omezení nebo problémy, které momentálně nemají řešení, nebo jsou inherentní součástí designu. Každý problém vysvětlujeme včetně dopadu a možných workaroundů.

### Problém 1: [Název]

[Popis problému, proč vzniká, jaký má dopad, případný workaround]

---

## 🔗 SOUVISLOSTI

[POPISNÝ TEXT vysvětlující, jak tato komponenta souvisí s ostatními částmi systému]

**Navazující dokumenty:**
- [base/related_doc.md](./related_doc.md) - Krátký popis co je v tom dokumentu
- [decisions/related_decision.md](../decisions/related_decision.md) - Krátký popis

**Implementováno v:**
- `path/to/code.py` - Krátký popis modulu

**Používá:**
- `path/to/dependency.py` - Krátký popis

---
