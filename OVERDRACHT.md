# OVERDRACHT — Kampbeheer (8ste FOS 't Vloedgat)

Dit document is voor de volgende beheerder van de kampbeheer-app. Het beschrijft hoe alles in elkaar zit, wat je jaarlijks moet doen, en wat je nooit mag vergeten. **Deze repo is publiek: zet hier nooit wachtwoorden, tokens of sleutels in.**

## 1. Wat is dit?

Eén webapp voor de kookverantwoordelijke van het kamp: menu's plannen, porties berekenen, bestellijsten per leverancier genereren (Solucious, supermarkt, slager, bakker), budget opvolgen en bestellingen afvinken per leverdatum.

- **Live site:** https://kamp2026.planelektra.be (= `index.html`)
- **Testomgeving:** https://kamp2026.planelektra.be/test.html (aparte testdata, wijzigingen daar raken het echte kamp niet)
- **Hosting:** GitHub Pages, vanuit deze repo (branch `main`). Gratis, geen server om te onderhouden.
- **Data:** Supabase (gratis tier), project-regio eu-west-1. Tabel `kampbeheer` met één rij per kamp (`kamp2026` = live, `test2026` = test), alles als één JSON-document in de kolom `data`.

## 2. Toegang die je nodig hebt (regel dit bij de overdracht!)

1. **GitHub-account** met schrijfrechten op deze repo (of eigenaarschap). Zonder dit kun je de site niet aanpassen.
2. **Supabase-account** met toegang tot het project. Zonder dit kun je niet aan de ruwe data of backups.
3. **Het app-wachtwoord** (mondeling doorgeven, staat nergens genoteerd).
4. **Solucious-login** van de groep (voor bestellingen en facturen).

**Advies:** hang GitHub en Supabase aan een groeps-mailadres (bv. een leiding@-adres) in plaats van aan een persoonlijk account. Dan vertrekt de toegang niet mee met een persoon.

## 3. Het app-wachtwoord wijzigen

Het wachtwoord staat niet in de code; alleen een SHA-256-hash ervan (constante `_kh` in `index.html` en `test.html`).

1. Kies een nieuw wachtwoord.
2. Genereer de hash: open de browser-console (F12) op de site en voer uit:
   `crypto.subtle.digest('SHA-256', new TextEncoder().encode('NIEUWWACHTWOORD')).then(b => console.log([...new Uint8Array(b)].map(x => x.toString(16).padStart(2,'0')).join('')))`
3. Vervang de waarde van `_kh` in **beide** bestanden (`index.html` en `test.html`) door de nieuwe hash en push.

**Let op:** dit is een UI-slot, geen echte beveiliging. Wie de broncode leest kan de databank rechtstreeks benaderen (zie §8).

## 4. Hoe wijzig je iets aan de app? (test → live)

Alle code zit in één HTML-bestand. Werkwijze:

1. **Wijzig eerst `test.html`**, push, en bekijk het resultaat op de test-URL.
2. Pas als het goed is: bouw `index.html` uit `test.html` met exact twee verschillen:
   - `const SB_ID='test2026';` → `const SB_ID='kamp2026';`
   - Verwijder de rode 🧪 TESTOMGEVING-bannerregel (één `<div>` bovenaan de body).
3. Push naar `main`. GitHub Pages deployt automatisch.

**Bekende storing:** de Pages-deploy faalt soms terwijl de push slaagt. Controleer na elke push het tabblad **Actions** in GitHub: de run "pages build and deployment" moet groen zijn. Faalt hij? Push een lege commit (`git commit --allow-empty -m "retrigger"`) en check opnieuw. Herhaal tot groen. **Een geslaagde push betekent NIET dat de site bijgewerkt is.**

Zie je je wijziging niet? Hard refresh (Ctrl+Shift+R) — de browser cachet de pagina.

## 5. Data: hoe gaat er nooit iets verloren?

Er zijn drie lagen. Gebruik ze alle drie.

1. **Handmatige export (belangrijkste!):** de knop **↓ backup** rechtsboven in de app downloadt alles als JSON-bestand. Doe dit **wekelijks tijdens de voorbereiding, dagelijks tijdens het kamp, en altijd vóór een grote wijziging**. Bewaar de bestanden op de gedeelde Drive van de groep, niet alleen op je eigen laptop. Terugzetten kan met **↑ herstel**.
2. **Automatische serverbackup:** een dagelijkse job (pg_cron in Supabase) kopieert de data naar de tabel `kampbeheer_backups`. Terugzetten vereist Supabase-toegang (SQL).
3. **Conflictbescherming:** als twee toestellen/tabbladen tegelijk werken, waarschuwt de app vóór het overschrijven en kun je kiezen: toch opslaan of herladen. Werk desondanks bij voorkeur met **één schrijvende gebruiker tegelijk**.

**Sluipend gevaar:** Supabase pauzeert gratis projecten na langdurige inactiviteit. Log buiten het kampseizoen af en toe in op het Supabase-dashboard, en zorg dat er na elk kamp een JSON-export op de Drive staat. Die export is je levensverzekering: daarmee kan de app altijd opnieuw opgebouwd worden, zelfs als Supabase volledig wegvalt.

## 6. Jaarritueel (elk nieuw kamp)

1. **Na afloop van het kamp:** exporteer een JSON-backup en zet die op de Drive.
2. **Nieuw kamp starten:** onderaan Setup → "Nieuw kamp starten". Het oude kamp wordt gearchiveerd (alleen-lezen, raadpleegbaar via 🗄 Archief) en het nieuwe start als kopie met verschoven datums. Er wordt eerst automatisch een backup gedownload en je moet NIEUW typen ter bevestiging.
3. **Setup bijwerken:** datums, takken, ledenaantallen, kampprijzen, terreinhuur.
4. **Prijzen verversen:** Solucious-prijzen verouderen. Na je eerste bestelling: importeer de bestelling/factuur in de app zodat de artikelprijzen actueel worden. Handmatige prijzen (het €/pers-veld bij recepten) vul je **inclusief btw** in; Solucious-prijzen worden **exclusief btw** opgeslagen en de app telt er zelf 6% bij.
5. **Leverdata instellen:** Kampkost → Bestellingen → leverdata invullen zodra Solucious de leverdagen bevestigt.
6. **Domein:** kamp2026.planelektra.be verwijst via het bestand `CNAME` en een DNS-record. Voor een nieuw jaar kun je het domein zo laten (de naam is cosmetisch) of aanpassen — dan moet zowel `CNAME` als het DNS-record mee.

## 7. Eigenaardigheden die je moet kennen

- **Btw:** Solucious toont in het winkelmandje prijzen **exclusief** btw; die sla je zo op en de app rekent ×1,06. Het handmatige €/pers-veld bij recepten rekent **geen** btw bij — daar vul je het inclusief-bedrag in.
- **"geen prijs" bij een ingrediënt** betekent meestal dat het ingrediënt geen gekende koppeling naar een Solucious-artikel heeft, of dat er nog geen prijs geïmporteerd/ingevuld is. Oplossen: prijs handmatig invullen in het recept, of de koppeling (laten) toevoegen in de code.
- **Gesponsorde gerechten:** vink "Gesponsord gerecht" aan in een recept en duid aan wat je zélf koopt; de rest telt op €0 en verschijnt als "niet bestellen" in de bestellijsten. Let op: gesponsord aanzetten zonder iets als "zelf" aan te vinken zet het hele gerecht op €0.
- **De 3% jeugdkampkorting** van Solucious (zomerperiode) zit **niet** in de berekeningen; de reële kost ligt dus iets lager dan de app toont.
- **Bestellijsten rekenen live** uit menu × aanwezigen. Wijzig je het menu of de aantallen, dan schuiven de hoeveelheden mee — maar reeds geplaatste afvinkjes worden gemarkeerd als "gewijzigd", controleer die dan opnieuw.

## 8. Beveiliging — eerlijk overzicht

- Het wachtwoordscherm is een drempel, geen slot: de API-sleutel in de broncode geeft lees- én schrijfrechten op de databank. Dit is een bewuste eenvoud-keuze. De bescherming tegen dataverlies zijn de backups (§5), niet het wachtwoord.
- Wil een technisch onderlegde opvolger dit ooit dichttimmeren: dat vereist echte server-side authenticatie (bv. Supabase Auth met aangescherpte row-level security) — een verbouwing, geen kleine fix. Weeg af of dat de complexiteit waard is.
- Deel nooit tokens/sleutels via chat of e-mail; roteer een sleutel meteen als dat toch gebeurd is.

## 9. Als alles stuk is (rampenplan)

1. **Site doet het niet, data intact:** check GitHub Actions (deploy groen?), forceer een redeploy (§4). DNS-probleem? Het domein wijst naar GitHub Pages; de site is ook bereikbaar via het standaard github.io-adres van de repo.
2. **Data kapot of gewist:** herstel via **↑ herstel** met de recentste JSON-export. Geen export? Supabase → tabel `kampbeheer_backups` → recentste rij voor `kamp2026` → kopieer die `data` terug naar de rij in `kampbeheer` (SQL of tabel-editor).
3. **Supabase weg (project gepauzeerd/verwijderd):** maak een nieuw gratis Supabase-project, maak tabel `kampbeheer` (kolommen: `id text primary key`, `data jsonb`, `updated_at timestamptz`), zet de nieuwe project-URL en anon-sleutel in de code (constanten bovenaan het script), en laad de JSON-export via **↑ herstel**.
4. **GitHub weg:** de site is één HTML-bestand + logo. Elke host die statische bestanden serveert werkt (Cloudflare Pages, Netlify, …). Upload `index.html`, `logo.png` en desgewenst `test.html`, en pas het DNS-record aan.

## 10. Hulp vragen aan een AI-assistent

Deze app is grotendeels met AI-hulp (Claude) gebouwd en onderhouden. Een opvolger zonder programmeerkennis kan wijzigingen laten uitvoeren door een AI-assistent toegang te geven tot deze repo. Werkbare afspraken daarbij: wijzig eerst `test.html` en controleer zelf het resultaat vóór iets live gaat; vraag altijd om de deploy-status te verifiëren ná een push; en maak vóór elke datawijziging een backup. Dit document plus de bestaande code bevatten alle context die zo'n assistent nodig heeft.
