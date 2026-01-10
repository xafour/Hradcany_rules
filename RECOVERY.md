# 🔧 RECOVERY SCÉNÁŘE

**Verze:** 1.0.0  
**Datum:** 2026-01-09  

### **Scénář #1: Token limit během chatu**

#### Prevence:
AI sleduje využití tokenů a varuje:
- 80% kapacity: "Blížím se k limitu"
- 90% kapacity: "Doporučuji commit současného stavu"

#### Když už došly tokeny:
1. Člověk: Začne nový chat s informací, že předchzí nebyl dokončen kvůli "Token limit"
2. Člověk: Zkopíruje poslední commit message do chatu
3. Člověk (volitelně): Ukáže git diff:
   `git diff HEAD~1 recognize_stamp.py`
   → Zkopíruje relevantní části do chatu
4. AI: "Pochopil jsem kde jsme skončili, pokračujeme"

### **Scénář #2: Zjištění ztráty kontextu**
1. Člověk: "Tohle jsme řešili v chatu #15"
2. AI: STOP všechno kódování
3. Člověk poskytne výsledek: `git log --grep="[klíčové slovo]"`
4. AI: Přečte a resumé: "Rozuměl jsem, je to XYZ"
5. Pokračujeme

### **Scénář #3: Dokument byl erodován**
1. Zjistíme že chybí důležité info
2. Člověk spustí: `git log -p <soubor>` → najde kde bylo smazáno
3. Obnová chybějící kontext
4. Append rozhodnutí proč bylo obnoveno
5. Commit: "RESTORE: <soubor> missing sections"
