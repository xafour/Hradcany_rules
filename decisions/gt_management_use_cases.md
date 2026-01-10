# GT Management - Use Cases Koncept

**Tags:** `#gt-management` `#use-cases` `#design`  
**Verze:** 1.0.0  
**Datum:** 2026-01-09  
**Status:** Active

---

## 🎯 CO JSOU USE CASES

Use Cases (UC) jsou základní stavební kameny GT Management systému. Každý Use Case popisuje jeden konkrétní scénář, jak uživatel nebo expert interaguje se systémem - co chce udělat, jak to udělá a co se přitom stane. Use Case není technická implementace, ale popis toho, JAK má systém fungovat z pohledu uživatele.

GT Management systém má celkem 17 Use Cases rozdělených do 4 workflows podle typu aktivity a uživatelské role. Každý Use Case řeší jeden konkrétní úkol - například "nahrát novou známku", "potvrdit klasifikaci", "schválit scan pro Ground Truth". Toto rozdělení nám umožňuje postupně implementovat systém po malých, testovatelných částech, a zároveň mít jasnou představu o tom, jak bude celý systém fungovat až bude dokončený.

---

## 🗂️ ČTYŘI WORKFLOWS

GT Management je organizován do čtyř hlavních workflows podle typu činnosti. Každý workflow sdružuje související Use Cases, které dohromady tvoří ucelený proces.

### WF-1: User Upload & Analyze

Tento workflow popisuje základní cestu nového skenu do systému. Začíná tím, že běžný uživatel nahraje fotografii nebo sken známky, kterou vlastní. Systém známku automaticky analyzuje - detekuje rámeček pomocí YOLO, vypočítá embeddings, pokusí se určit denomination. Ale protože automatická detekce není stoprocentní, metadata (typ známky, tisková deska, pozice) zatím neukládáme jako potvrzené. Sken je ve stavu "pending" a čeká na potvrzení. Teprve po potvrzení klasifikace uživatelem nebo expertem se sken může stát součástí Ground Truth.

**Use Cases:**
- **UC-1: Upload & Analyze** - Uživatel nahraje známku, systém ji analyzuje a navrhne klasifikaci
- **UC-2: Confirm Classification** - Uživatel nebo expert potvrdí správnost klasifikace

Tento dvoufázový proces (upload → confirm) zajišťuje, že do Ground Truth se dostanou pouze ověřené, správně klasifikované známky.

---

### WF-2: Expert Review & Verification

Tento workflow popisuje pokročilé činnosti, které může provádět pouze expert. Expert má na starosti kvalitu Ground Truth - schvaluje nové skeny, které čekají na potvrzení, odmítá špatně naskenované nebo chybně klasifikované známky, a pravidelně kontroluje kvalitu již schválených skenů.

**Use Cases:**
- **UC-5: Expert Approval** - Expert schvaluje pending skeny do GT (jednotlivě, automaticky nebo hromadně)
- **UC-6: Expert Rejection** - Expert odmítá špatné skeny nebo je reklasifikuje
- **UC-7: Quality Review** - Expert kontroluje kvalitu GT, flaguje podezřelé skeny, demotuje špatné skeny zpět do pending

Expert má větší pravomoci než běžný uživatel. Může schvalovat cizí uploady, odmítat skeny, měnit klasifikaci již schválených známek v Ground Truth. Toto je záměrný design - kvalita Ground Truth je kritická pro přesnost recognition systému, proto je potřeba mít roli s dostatečnými pravomocemi k jejímu udržování.

---

### WF-3: User Management

Tento workflow popisuje, jak běžný uživatel spravuje své vlastní uploady. Každý uživatel vidí POUZE své vlastní nahrané skeny - cizí uploady nevidí (privacy-first design). Uživatel si může prohlédnout svoje skeny, filtrovat je podle stavu (pending/verified/rejected), upravovat klasifikaci u pending skenů, nebo smazat svoje uploady ze svého view.

**Use Cases:**
- **UC-3: View My Uploads** - Uživatel vidí pouze svoje uploady s možností filtrování
- **UC-4: Batch Import** - Uživatel nahraje více skenů najednou (API, CSV, streaming)
- **UC-17: Delete from View** - Uživatel smaže sken ze svého view (multi-user safe)

Privacy-first design znamená, že uživatel A nikdy nevidí, co nahrál uživatel B. Expert vidí pending queue všech uživatelů (protože je musí schvalovat), ale běžný uživatel vidí jen svoje. Toto pravidlo platí i pro duplicity - pokud dva uživatelé nahrají stejný sken (detekováno pomocí SHA256), každý vidí "svůj" upload, ale v storage je uložen pouze jeden soubor.

---

### WF-4: Expert Queues

Tento workflow popisuje, CO expert vidí, když se přihlásí do systému. Expert má přístup ke třem frontám: pending queue (skeny čekající na schválení od všech uživatelů), suspect queue (skeny označené jako problematické), a vlastní uploady (skeny které expert sám nahrál).

Expert NEVIDÍ celý Ground Truth katalog jako browsable databázi. Ground Truth je interní know-how recognition systému, ne veřejný katalog. Expert může Ground Truth upravovat (přes UC-7 Quality Review), ale neuvidí stránkovatelný seznam "všech známek v GT".

Toto rozhodnutí jsme učinili proto, že Ground Truth obsahuje tisíce skenů a jeho browsing by byl neefektivní. Expert potřebuje vidět co vyžaduje jeho pozornost (pending, suspect), ne všechno.

---

## 👥 UŽIVATELSKÉ ROLE

GT Management rozlišuje tři role s různými pravomocemi:

### Běžný uživatel (Regular User)

Běžný uživatel je sběratel nebo filatelista, který chce analyzovat svoje skeny. Může nahrávat svoje skeny (UC-1), potvrzovat nebo opravovat navrhovanou klasifikaci (UC-2), prohlížet si svoje uploady (UC-3), a mazat svoje skeny ze svého view (UC-17).

Běžný uživatel NEVIDÍ cizí uploady, ani nemůže schvalovat skeny do Ground Truth. Jeho uploady zůstávají ve stavu "pending" dokud je neschválí expert nebo dokud systém neaplikuje auto-approval (pokud jsou splněny podmínky - certifikát + vysoká confidence).

### Expert

Expert je důvěryhodný uživatel s filatelistickými znalostmi, který má na starosti kvalitu Ground Truth. Má všechny pravomoci běžného uživatele plus navíc může: schvalovat pending skeny do GT (UC-5), odmítat nebo reklasifikovat skeny (UC-6), kontrolovat kvalitu GT a flagovat podezřelé skeny (UC-7).

Expert vidí pending queue všech uživatelů (ne jen svoje) a suspect queue. Může upravovat metadata již schválených skenů v GT a demotovat skeny z GT zpět do pending, pokud zjistí chybu.

### Superuser (budoucí rozšíření)

Superuser je administrátor systému s maximálními pravomocemi. Může dělat všechno co expert plus navíc spravovat uživatelské účty, konfigurovat pravidla auto-approval, a provádět hromadné operace přes administrátorské rozhraní. Role superuser zatím není v Use Cases implementovaná, je plánovaná pro budoucí rozšíření. V současnosti je prováděna vlastníkem systému.

---

## 🔄 TYPICKÝ ŽIVOTNÍ CYKLUS SKENU

Popis jak prochází sken systémem od uploadu po Ground Truth:

### 1. Upload (UC-1)

Uživatel nahraje fotografii známky přes webové rozhraní. Systém detekuje rámeček pomocí YOLO, warp známku na standardizovaný formát 1300×1100, vypočítá embeddings a porovná je s referenčními embeddingy. Na základě této analýzy navrhne classification - například "500h TD I ZP 42" s confidence HIGH. 

V databázi se vytvoří záznam v tabulce `reference_front` s `confirmed=0` (pending). Metadata `stamp_type_id`, `plate_id`, `zp_no` jsou zatím NULL (nepotvrzené). V tabulce `user_uploads` se zaznamená vlastnictví - kdo tento sken nahrál. Originální soubor se uloží do SHA-based storage. YOLO quad se cachuje v `inference_frames` pro případnou budoucí re-analýzu.

### 2. Pending stav

Sken je nyní v pending queue. Uživatel ho vidí ve svém seznamu uploadů se stavem "čeká na potvrzení". Expert ho vidí v pending queue všech uživatelů. Systém zobrazuje navrhovanou klasifikaci (např. "500h TD I ZP 42, confidence HIGH") a čeká na potvrzení.

V tomto stavu je sken kompletně analyzovaný (má embeddings, YOLO cache), ale ještě není součástí Ground Truth. Recognition pipeline ho nepoužívá při matchingu nových skenů. Můžeme ho kdykoliv reklasifikovat nebo smazat bez dopadu na systém.

### 3. Confirm (UC-2)

Uživatel nebo expert potvrdí klasifikaci. Pokud souhlasí s navrženou klasifikací, stiskne "Confirm". Pokud chce opravit, může změnit denomination, tiskovou desku nebo pozici a pak teprve potvrdit.

Systém updatuje záznam v `reference_front`: vyplní `stamp_type_id`, `plate_id`, `zp_no` podle potvrzené klasifikace, a nastaví `confirmed=1` (verified). Pak spustí výpočet embeddingů - vytvoří všechny required crops pro danou drawing_id a vypočítá embeddings, které se uloží do `reference_embeddings` s `is_reference=0` (pending GT approval).

### 4. Auto-approval nebo Expert Review

Pokud sken splňuje podmínky pro auto-approval (má certifikát, high confidence, OCR match), systém automaticky nastaví `confirmed=1` a embeddings jako `is_reference=1` (aktiv v GT). Sken se stává okamžitě součástí Ground Truth.

Pokud podmínky nesplňuje, čeká na expert review (UC-5). Expert sken zkontroluje a buď schválí (approve → `confirmed=1`, embeddings `is_reference=1`), nebo odmítne (reject → `confirmed=-1`, embeddings se smažou).

### 5. Ground Truth

Sken je nyní součástí Ground Truth. Má `confirmed=1` a jeho embeddings jsou `is_reference=1`, což znamená že recognition pipeline je používá při matchingu nových skenů. Sken je stále vlastněn původním uživatelem (v `user_uploads`), ale expert ho může upravovat nebo v případě zjištění chyby demotovat zpět do pending.

### 6. Quality Review (volitelné)

Periodicky (například každých 30 dní) systém automaticky flaguje GT skeny pro quality review. Expert je zkontroluje (UC-7) a buď: potvrdí že jsou OK (clear suspect flag), nebo najde problém a demotuje zpět do pending, nebo kompletně odstraní z GT.

Tímto se zajišťuje že Ground Truth neobsahuje zastaralé nebo chybné skeny.

---

## 🔑 KLÍČOVÁ ROZHODNUTÍ

### Privacy-first design

Rozhodli jsme se, že běžný uživatel vidí POUZE svoje uploady. Toto rozhodnutí jsme učinili z několika důvodů: (1) ochrana soukromí - uživatel A nemá vidět co uploaduje uživatel B, (2) jednoduchost UI - uživatel není zahlcený cizími skeny, (3) motivace k přispívání - každý vidí svůj vlastní progress.

Technicky je toto řešeno tabulkou `user_uploads`, která ukládá vztah scan_id ↔ user. Dotazy na seznam skenů vždy filtrují podle `uploaded_by = current_user`. Expert má výjimku - vidí pending queue všech uživatelů, protože je musí schvalovat.

### Dvoufázový workflow (Upload → Confirm)

Rozhodli jsme se oddělit nahrání skenu (UC-1) od potvrzení klasifikace (UC-2). V UC-1 systém analyzuje a navrhuje classification, ale metadata zůstávají NULL (nepotvrzené). Teprve v UC-2 se metadata potvrdí a embeddings se vypočítají.

Důvod: Automatická detekce není stoprocentní. Pokud bychom okamžitě při uploadu vyplnili metadata a vypočítali embeddings, riskovali bychom že do Ground Truth se dostanou chybně klasifikované skeny. Dvoufázový proces dává uživateli možnost zkontrolovat a opravit navrhovanou klasifikaci před tím, než se stane součástí GT.

### Ground Truth jako interní know-how

Rozhodli jsme se že Ground Truth NENÍ browsable katalog, ale interní know-how recognition systému. Expert nevidí stránkovatelný seznam "všech 10,000 známek v GT", ale vidí pouze co vyžaduje jeho pozornost: pending queue, suspect queue, vlastní uploady.

Důvod: Ground Truth může obsahovat desítky tisíc skenů a jeho browsing by byl neefektivní. Expert nepotřebuje procházet všechny známky, potřebuje řídit kvalitu - schvalovat nové, kontrolovat podezřelé. Pokud někdy v budoucnu budeme chtít GT publikovat jako katalog, bude to samostatná funkce s read-only přístupem.

### SHA256 deduplication

Rozhodli jsme se používat SHA256 hash originálu jako primární identifikátor skenu. Když dva uživatelé nahrají stejný sken (detekováno pomocí SHA256), v databázi budou dva záznamy v `user_uploads`, ale v storage pouze jeden soubor.

Důvod: (1) úspora místa - stejný sken se neukládá dvakrát, (2) konzistence - pokud najdeme chybu v jednom skenu, chyba se automaticky projeví u všech, kdo ten sken uploadovali, (3) multi-user safe - každý uživatel "vlastní" svůj upload (má ho ve svém seznamu), i když fyzicky je to stejný soubor.

### Nullable metadata v pending stavu

Rozhodli jsme se že v pending stavu jsou metadata (`stamp_type_id`, `plate_id`, `zp_no`) nullable (NULL) místo použití placeholder hodnot.

Důvod: NULL přirozeně reprezentuje stav "neznámé" nebo "čeká na potvrzení". Kdyby jsme použili placeholder (např. stamp_type_id=0), museli bychom rozlišovat mezi "skutečnou známkou s ID 0" a "placeholder". NULL je čistší řešení. Viz `decisions/schema_v3_1_0_nullable_columns.md` pro detaily.

---

## 🔗 SOUVISLOSTI

**Technická implementace:**
- [base/gt_workflows.md](../base/gt_workflows.md) - Detailní workflow diagramy
- [base/database_schema.md](../base/database_schema.md) - DB tabulky pro GT Management

**Další rozhodnutí:**
- [decisions/uc1_workflow.md](./uc1_workflow.md) - Detailní popis UC-1
- [decisions/schema_v3_1_0_nullable_columns.md](./schema_v3_1_0_nullable_columns.md) - Proč nullable
- [decisions/privacy_first_design.md](./privacy_first_design.md) - Proč privacy-first (když bude vytvořen)

**Implementační status:**
- [tasks/uc1_implementation.md](../tasks/uc1_implementation.md) - UC-1 status
- [tasks/bug_uc1_file_storage.md](../tasks/bug_uc1_file_storage.md) - Známý bug

---

**Poslední aktualizace:** 2026-01-09  
**Autor:** Milan + Claude  
**Status:** ✅ ACTIVE

---
