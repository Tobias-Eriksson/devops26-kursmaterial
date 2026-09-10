# M2 – Branch protection, buddy-review och en riktig konflikt

Den här labben hör till **session 2** (Versionshantering).

**Milstolpens bevis:** branch protection är på i ert repo, var och en har en
mergad pull request med godkänd review (solo: med dokumenterad
självgranskning), historiken innehåller en löst
merge-konflikt, och taggen `m2-review` är uppe.

Från och med idag är det här **kursens normala arbetssätt**: allt som går in
i `main` går via en granskad pull request. M3–M9 ändrar bara *vad* som går
genom flödet — aldrig *hur*.

## Steg 0 – Förkrav

- [ ] Uppdatera kursmaterialet: `cd course-material && git pull`.
- [ ] Öppna er codespace (eller er lokala klon) — labben nedan körs i en
      terminal där.
- [ ] M1 är klar: repot finns, taggen `m1-repo` är uppe. Inte klar? Gör
      klart M1 först (`labs/m1-forsta-commit.md`) — M2 sker i samma repo.
- [ ] **Par:** ni jobbar i ert gemensamma repo och är varandras
      review-buddies. Båda måste ha accepterat collaborator-inbjudan.
- [ ] **Solo:** du jobbar helt själv i ditt eget repo — ingen
      review-buddy, ingen collaborator att bjuda in. GitHub låter aldrig
      en PR-författare approva sin egen PR, så solo-vägen sätter
      **Required approvals** till **0** i steg 1 och granskar sin egen PR
      i steg 4 (dokumenterad självgranskning). Allt annat i labben är
      exakt detsamma.

## Steg 1 – Slå på branch protection

På GitHub, i ert repo:

1. **Settings** → **Rulesets** → **New ruleset** → **New branch ruleset**.
2. Namnge regeln (t.ex. `Main branch protection`) och under **Target
   branches**: **Add target** → **Include default branch** (träffar `main`).
3. Kryssa i **Require a pull request before merging** — den gäller
   alla, så direktpush till `main` avvisas oavsett om ni är ett par eller
   jobbar solo (steg 2 testar just det). Under den: **Required
   approvals** — **1 om ni är ett par**, **0 om du jobbar solo**.
   Varför 0 för solo? GitHub låter aldrig PR-författaren approva sin egen
   PR, så med kravet 1 och ingen collaborator skulle du inte kunna merga
   alls. PR-tvånget står kvar — granskningskravet flyttas bara från
   GitHub till dig själv, i steg 4.
4. Högst upp på sidan: sätt **Enforcement status** till **Active**. Nya
   rulesets skapas som **Disabled** — regeln finns men gör ingenting förrän
   ni gör det här steget. Lätt att missa, så dubbelkolla innan ni sparar.
5. **Create**.

Bypass-listan är tom som standard i en ny ruleset, så regeln gäller redan
**er själva** — till skillnad från classic branch protection är inte
repo-ägaren automatiskt undantagen. Inget extra kryssrutssteg behövs för
det.

Låt listan vara tom. Att lägga in sig själv där är **inte** lösningen på
en merge som är blockerad — då slutar skyddet gälla för allt annat också.
Kan du inte merga: kolla antalet under *Required approvals* (0 för solo)
och steg 2:s diagnos.

Kryssa **inte** i *Require status checks to pass before merging* ännu.
Checken `Lint and test backend` (CI-workflowen som följde med från
template-repot) syns redan på varje PR ni öppnar — men **obligatorisk** gör
vi den först i M5, när ni byggt en pipeline själva och vet vad den gör.
Ert repos egen `README.md` beskriver det steget, för den nyfikna.

> 📸 **Kom ihåg skärmdump till inlämningen:** inställningssidan för regeln, med **Enforcement status: Active** och **Required approvals** synligt.

## Steg 2 – Testa skyddet

Övertyga er om att låset fungerar. Stå på `main` och försök pusha direkt:

```bash
git switch main
git pull
git commit --allow-empty -m "test: try pushing directly to main"
git push
```

Pushen ska **avvisas** — `remote rejected ... protected branch` eller
*"Changes must be made through a pull request"*. Det är inte ett fel, det
är regeln ni just satte.

**Avvisas pushen inte — går den bara igenom?** Kolla två saker i tur och
ordning:

1. **Repot är Private på ett gratis GitHub-konto.** Regler enforce:as
   inte alls där (GitHub visar en banner om det på inställningssidan för
   regeln). Snabbast: byt till Public: **Settings** → **General** →
   **Danger Zone** → **Change visibility** → **Public**. Vill ni hellre
   behålla repot Private? Gå med i **GitHub Education**
   (https://github.com/education) — då enforce:as regler även på
   Private-repon. Godkännandet kan ta tid, så Public är alternativet
   medan labben pågår.
2. **Enforcement status står kvar på Disabled.** Vanligaste fällan: steg
   1 punkt 4 glömdes bort. Öppna **Settings** → **Rulesets** → regeln och
   sätt **Enforcement status** till **Active**. (GitHub visar ibland en
   annan vy — skapade ni av misstag en classic-regel i stället för en
   ruleset funkar den också, men resten av labben och M5 förutsätter
   ruleset-vyn, så byt tillbaka om ni kan.)

Testa om (samma kommando som ovan) efter fixen. Städa sedan bort
testcommitten (den var tom, inget går förlorat):

```bash
git reset --hard origin/main
```

> 📸 **Kom ihåg skärmdump till inlämningen:** terminalen med raden där pushen avvisas.

## Steg 3 – Var sin pull request

Nu den enda vägen in i `main`. **Var och en i paret** tar ett uppdrag —
olika filer, så ni inte krockar *ännu* (krocken kommer i steg 5, under
kontrollerade former). **Solo:** välj **ett** av uppdragen.

Båda uppdragen har en **uttalad spec**: reviewern ska kunna kontrollera
att den är uppfylld, inte tycka till om smaken.

### Uppdrag A — hämta ett item (backend)

Ny endpoint `GET /api/items/{item_id}` i `backend/app/main.py`:

- finns itemet: returnera det
- finns det inte: svara **404** med `Item not found` — samma mönster som
  `delete_item` några rader längre ner i filen, läs den först
- **två nya tester** i `backend/tests/test_main.py`, skrivna på samma sätt
  som de som redan finns där:
  - *träffen*: skapa ett item med `client.post(...)` (som i
    `test_create_and_list_item`), hämta det med
    `client.get(f"/api/items/{created['id']}")` och kontrollera att
    status är **200** och att `text` och `id` i svaret är samma som du
    skapade
  - *404:an*: `client.get("/api/items/999999")` ska ge status **404**
    (som i `test_delete_missing_item_returns_404`)
- branch: `add-get-item`

Ungefär sex rader kod plus två korta tester. `cd backend && pytest` ska
vara grön innan du pushar.

### Uppdrag B — färgtema (frontend)

Ett färgtema som **CSS-variabler**, bara i `frontend/style.css`:

- deklarera dem en gång överst i filen, i en `:root`-regel:
  `:root { --bg: #f4f1ea; --fg: #222; --accent: #b3541e; }` — namnen och
  färgerna är fria, det här visar bara syntaxen
- använd dem med `var(...)`: `body` får `background: var(--bg)` **och**
  `color: var(--fg)`, `button` får `background: var(--accent)`
- den hårdkodade kanten `1px solid #ddd` på `li` ska använda en variabel i
  stället (`var(--accent)`, eller lägg till en egen `--border` i `:root`)
- inga hårdkodade färger kvar utanför `:root` — de enda `#`-värdena i filen
  ska stå i `:root`-regeln
- branch: `add-color-theme`

Ungefär tio rader. Kontrollera själv innan du pushar: `docker compose up
--build -d`, öppna port **8080** (codespacen forwardar den och visar en
*Open in Browser*-knapp) — texten ska vara läsbar mot den nya bakgrunden
och Add-knappen synlig. `docker compose down` när du är klar. Vad
`docker compose` egentligen *gör* är nästa veckas föreläsning (M3) — i dag
räcker det att kommandot startar appen.

**Rör inte** de här, de behövs orörda senare: `/api/health` (S4 och
driftstestet i M9 asserterar exakt dess svar), `<h1>`-rubriken i
`frontend/index.html` (steg 5 bygger sin konflikt på just den raden) och
allt som räknar antalet notes (M4:s buggjakt utgår från det).

```bash
git switch main && git pull        # utgå alltid från färskaste main
git switch -c add-get-item         # uppdrag B: add-color-theme (även fix/... funkar) — döp efter vad den gör
# ...gör ändringen (se uppdraget ovan)...
git add backend/app/main.py backend/tests/test_main.py   # uppdrag B: git add frontend/style.css
git commit -m "Add endpoint for fetching a single item"  # varför, inte bara vad
git push -u origin add-get-item
```

Öppna PR:en: GitHub visar en gul banner *"Compare & pull request"* — eller
gå till **Pull requests** → **New**. Skriv en mening om **vad** och
**varför** — och lägg till ett kort avsnitt **"Så testar du"**: 2–3 rader
med kommandot eller klicket, och vad reviewern ska se. Till exempel:

```text
## Så testar du
cd backend && pytest
Alla tester ska passera, inklusive de två nya (träff + 404 för okänt id).
```

Välj sedan din buddy under **Reviewers** i högerspalten.

**Solo:** hoppa över *Reviewers* — men "Så testar du" är extra viktigt för
dig: det är ingången till självgranskningen i steg 4.

### Alternativ: låt ett AI-verktyg göra ändringen

Uppdrag A och B får göras **för hand eller med ett AI-verktyg** — ditt val,
och review-reglerna är exakt desamma. Det gäller både par och solo. Vinsten
med AI-vägen är inte sparad tid: du får granska kod du **inte skrev själv**,
och det är precis den muskeln reviewn tränar.

Alla tre CLI:er finns i devcontainern (Claude Code, Codex, Gemini). Codex har
en gratisnivå (per 9.9.2026) — den som har en egen prenumeration på Codex,
Gemini eller Claude kan använda den i stället. Starta Codex med `codex` i
terminalen och logga in när det frågar.

Fyra regler när verktyget gör jobbet:

1. **Ny branch från färsk `main` — och bara uppgiftens filer i commiten.**
   kontrollera diffen på PR:en: ligger något annat med, be verktyget ta
   bort det ur commiten.
2. **PR-texten verktyget skriver är ett påstående, inte ett bevis.** Kör
   själv `cd backend && ruff check . && pytest` (uppdrag A) eller
   `docker compose up --build -d` + port **8080** (uppdrag B) innan du
   kommenterar och mergar — och skriv **"Så testar du"** utifrån vad *du*
   körde och såg, inte vad verktyget säger att det körde.
3. **Be aldrig verktyget radera eller force-pusha en branch som redan är
   pushad.** Det behövs inte här, och det är just den sortens kommando som
   kan radera någon annans arbete. Blev något fel: gör en ny commit på samma
   branch, PR:en uppdateras av sig själv.
4. **Räkna med godkännande-frågor.** Codex frågar innan det kör
   git-kommandon — läs vad det vill köra och godkänn ett kommando i taget.
   Det är också en review, bara i terminalen.

**PR-ägaren är alltid du.** `gh` använder din egen GitHub-inloggning i
codespacen, så PR:en står i ditt namn — ett AI-verktyg kan därför aldrig vara
den "andra granskaren". Uppställningen är alltså oförändrad: **solo** har
*Required approvals* **0** och gör den dokumenterade självgranskningen i steg
4, **par** låter partnern granska som vanligt.

Ser du `.devcontainer/devcontainer-lock.json` som ändrad i `git status`:
det är codespacen som skrivit om filen (en radbrytning) — ta med den i
din första PR, det händer bara en gång.

## Steg 4 – Review och merge

Reviewern öppnar PR:en → fliken **Files changed** och går igenom
checklistan (samma som på föreläsningens slide):

- [ ] Förstår jag ändringen — oavsett vem eller vad som skrev den? Om inte —
      **fråga i en kommentar**
- [ ] Diffen innehåller bara det PR:en handlar om — inga
      överraskningsfiler (`.DS_Store`, hemligheter, tillfälliga filer) —
      utom `devcontainer-lock.json` första gången, se steg 3
- [ ] Commit-meddelandena säger *varför*, inte bara *vad*
- [ ] Checken `Lint and test backend` är grön — röd? Titta i loggen ihop
- [ ] Inga kvarglömda konfliktmarkörer (`<<<<<<<`) i diffen

Punkt 1 är den som avgör om reviewn är värd något — och den gäller lika mycket
**författaren**: läs din egen diff innan du öppnar PR:en, särskilt om du lät ett
AI-verktyg (Claude Code, Codex och Gemini finns i devcontainern) skriva den — se
*Alternativ: låt ett AI-verktyg göra ändringen* i steg 3. AI genererar — ni
granskar som människor.

**Att läsa diffen räcker inte i dag — checka ut branchen och kör den.**

Uppdrag A:

```bash
git fetch origin
git switch add-get-item        # eller: gh pr checkout <PR-nummer>
cd backend && pytest
```

Grön? Kontrollera också att testerna täcker **båda** fallen — träffen
*och* 404:an, inte bara det som fungerar.

Uppdrag B:

```bash
git fetch origin
git switch add-color-theme
docker compose up --build -d   # vad kommandot gör: M3, nästa vecka
```

Öppna port **8080** (codespacen visar en *Open in Browser*-knapp): är
texten läsbar mot den nya bakgrunden, syns Add-knappen? Och i diffen: inga
hårdkodade färger kvar utanför `:root`, bara `style.css` ändrad. Städa efter dig:

```bash
docker compose down
git switch main
```

Går det inte att verifiera utifrån PR-beskrivningens **"Så testar du"** —
saknas kommandot, eller stämmer det inte? Då är det din **första
kommentar**.

Kommentera på en rad: håll muspekaren över radnumret i diffen och klicka
det blå plusset. Lämna **minst en** radkommentar på något du faktiskt
kontrollerade. Något att fixa? Författaren committar och pushar på
**samma branch** — PR:en uppdateras av sig själv, ingen ny PR behövs.

Godkänn: **Review changes** → **Approve** — och skriv en mening om vad du
faktiskt kontrollerade, inte att det ser bra ut: *"checkade ut branchen,
`pytest` grön, båda fallen täckta"* eller *"körde appen på 8080, texten
läsbar och knappen syns"*. Sedan mergar **författaren**: **Merge pull
request** (standardvalet *Create a merge commit*) → **Delete branch**.

**Par:** en PR som ett AI-verktyg skrev granskas precis som en handskriven —
samma checklista, samma utcheckning och körning. Och författaren svarar för
koden även då: *"AI:n skrev den"* är inget svar i en review.

**Solo — så granskar du din egen PR.** Du kan inte approva den (GitHub
tillåter det aldrig), och med *Required approvals* på **0** behöver du
inte. I stället gör du granskningen **synlig i PR:en**:

1. **Beskrivningens "Så testar du"** säger hur du testade ändringen —
   kommandot eller klicket, och vad du såg (t.ex. `cd backend && pytest`
   → grönt, båda fallen täckta).
2. **Checka ut branchen och kör den** — samma kommandon som ovan: `pytest`
   för uppdrag A, `docker compose up --build -d` + port 8080 för uppdrag
   B. Det är den delen av reviewn som inte går att fejka för sig själv.
3. Öppna **Files changed** och läs din egen diff mot checklistan ovan.
4. Lämna **minst en radkommentar** på något du faktiskt kontrollerade:
   plusset vid radnumret, sedan **Review changes** → **Comment** (inte
   *Approve* — den vägen är stängd på egen PR).
5. Merga: **Merge pull request** → **Delete branch**.

Lät du ett AI-verktyg göra ändringen (steg 3)? Flödet är exakt detsamma — och
det blir en riktigare review: diffen du läser är inte din egen. Kom ihåg att
**"Så testar du"** ska säga vad *du* körde, inte vad verktyget påstod att det
körde.

Beskrivningen och kommentarerna ligger kvar på PR:en — det är bevisspåret
som läses när milstolpen bedöms. Och var ärlig om vad solo-vägen tappar:
ingen annan läser din kod, så det du själv är blind för slinker igenom.
Allt annat står kvar — branch, PR, beskrivning som förklarar ändringen,
grön check, ingen direktpush till `main`.

Städa lokalt:

```bash
git switch main && git pull
git branch -d add-get-item      # eller add-color-theme
```

**Par:** byt roller och gör steg 4 en gång till för den andras uppdrag, så
att **båda** har en mergad PR med godkänd review. **Solo:** din mergade PR med dokumenterad
självgranskning räcker — gå vidare till steg 5.

> 📸 **Kom ihåg skärmdump till inlämningen:** PR-sidan med reviewkommentaren och godkännande-meningen (**solo:** din egen radkommentar i stället för ett godkännande).

## Steg 5 – Konfliktövningen

Nu framkallar vi en merge-konflikt **med flit** — så att första gången ni
ser markörerna är nu, tillsammans, och inte en sen kväll före deadline.
Receptet: båda utgår från **samma commit** och ändrar **samma rad**.

1. **Båda:** ställ er på samma utgångspunkt och kontrollera att ni ser
   **samma** översta commit:

   ```bash
   git switch main && git pull
   git log -1 --oneline
   ```

2. **Båda samtidigt** — skapa var sin branch och ändra **samma rad**,
   `<h1>`-rubriken i `frontend/index.html`, till olika värden
   (använd era riktiga namn):

   ```bash
   git switch -c konflikt-alice        # partner B: konflikt-bob
   ```

   Partner A sätter raden till `<h1>Alices anteckningar</h1>`,
   partner B till `<h1>Bobs anteckningar</h1>`. Committa och pusha,
   öppna var sin PR precis som i steg 3.

3. **Merga partner A:s PR först** (snabb review + approve räcker här).

4. Titta nu på **partner B:s PR**: GitHub visar *"This branch has
   conflicts that must be resolved"* — merge-knappen är låst. Precis som
   planerat. (Knappen *Resolve conflicts* öppnar en webbeditor — den
   fungerar, men vi löser i terminalen — i er codespace, eller lokalt om
   ni kör så: det är så det ser ut i verkligheten, och det är den
   färdigheten M2 tränar.)

5. **Partner B löser konflikten i sin arbetsmiljö** (codespacen eller
   lokalt) — hämta in nya `main` i branchen:

   ```bash
   git switch konflikt-bob
   git fetch origin
   git merge origin/main
   ```

   Git svarar `CONFLICT (content): Merge conflict in frontend/index.html`.
   Öppna filen och hitta frågan Git inte kunde besvara:

   ```text
   <<<<<<< HEAD
       <h1>Bobs anteckningar</h1>
   =======
       <h1>Alices anteckningar</h1>
   >>>>>>> origin/main
   ```

   `HEAD`-sidan är **din branch**, sidan under `=======` är det du mergar
   in (`origin/main`, där A:s ändring nu ligger). Välj inte — **kombinera**:
   ersätt hela blocket, markörerna inklusive, med en rad:

   ```html
       <h1>Alices och Bobs anteckningar</h1>
   ```

   Tala om för Git att frågan är besvarad:

   ```bash
   git add frontend/index.html
   git commit          # Gits färdiga förslag på meddelande räcker — spara och stäng
   git push
   ```

   (Panik i stället? `git merge --abort` ångrar hela mergen och du kan
   börja om från punkt 5.)

6. Ladda om PR-sidan: konflikten är borta och merge-knappen upplåst.
   Review + approve + merge + **Delete branch**, som vanligt. Kolla till
   sist att inga markörer glömts kvar någonstans:

   ```bash
   git switch main && git pull
   git grep '<<<<<<<'        # ska vara helt tyst
   ```

**Solo?** Kör exakt samma recept själv: två brancher (`konflikt-1`,
`konflikt-2`) från samma `main`-commit, ändra samma rad till olika värden,
öppna två PR:ar, merga den första direkt (självgranska den som i steg 4),
lös konflikten i den andra enligt punkt 5–6.

> 📸 **Kom ihåg skärmdump till inlämningen:** GitHubs "This branch has conflicts"-banner innan ni löser den, och/eller terminalens `CONFLICT`-rad.

## Steg 6 – Tagga milstolpen

```bash
git switch main && git pull
git tag m2-review
git push origin m2-review
```

**Bevisa det osynliga:** branch protection-inställningarna och testet där
pushen till `main` avvisades syns inte i repot. Ni har nu 4 skärmdumpar
från steg 1, 2, 4 och 5 — lägg dem i `inlamning/` och skriv några
meningar per skärmdump i `inlamning/m2-review.md` och committa, formatet
visas i `inlamning/m0-exempel.md`. **Solo:** nämn där att du körde med
*Required approvals* **0** och länka din PR — självgranskningen
(beskrivning + radkommentar) är det som ersätter buddyns godkännande.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] Branch protection är på: en push direkt till `main` avvisas (steg 2).
- [ ] **Par:** var och en har minst en mergad PR med **godkänd review**
      av buddyn (fliken *Pull requests* → *Closed*). **Solo:** en mergad
      PR med dokumenterad självgranskning — beskrivningen säger hur du
      testade, och minst en egen radkommentar ligger kvar på diffen.
- [ ] Konflikten är löst och mergad — `git grep '<<<<<<<'` på `main` är
      tyst, och rubriken i appen är den kombinerade.
- [ ] Mergade brancher är raderade på GitHub (*Branches*-sidan är ren).
- [ ] Taggen `m2-review` syns under **Tags** på GitHub.
- [ ] Det osynliga arbetet är dokumenterat i `inlamning/m2-review.md`.

## Vanliga problem

**"Min PR kan inte mergas" — grå merge-knapp, *Review required*.**
Branch protection gör sitt jobb: ingen godkänd review än. Be din buddy
granska. Har buddyn redan godkänt? Då räknades godkännandet inte — buddyn
har inte skrivåtkomst (collaborator-inbjudan inte accepterad, se steg 0)
eller så godkände du din egen PR (räknas aldrig). **Solo:** då står
*Required approvals* kvar på **1** — sätt den till **0** (steg 1). Lägg
inte till dig själv i bypass-listan i stället.

**"Min PR kan inte mergas" — *This branch has conflicts*.**
Det är konfliktövningens läge: någon annans ändring av samma rader har
mergats före dig. Lös enligt steg 5 punkt 5–6 — det fungerar likadant för
äkta, oplanerade konflikter.

**`git push` avvisas med `remote rejected ... protected branch`.**
Du försöker pusha direkt till `main` — det är stängt nu (det var steg 2:s
poäng). Gör ändringen på en branch och öppna en PR.

**`git push` går igenom, fast ni satte upp branch protection i steg 1.**
Se diagnosen i steg 2: antingen är repot **Private** på ett gratis
GitHub-konto (regler enforce:as inte alls där — byt till Public, eller gå
med i **GitHub Education** om ni vill behålla Private), eller så står
regelns **Enforcement status** kvar på **Disabled** — vanligast för att
steg 1 punkt 4 (sätt den till **Active**) glömdes bort.

**Checken `Lint and test backend` är röd på min PR.**
Röd check blockerar inte mergen i M2 (obligatorisk blir den först i M5) —
men röd betyder att lint eller test faktiskt failar. Klicka på checken →
läs loggen → fixa och pusha. Merga inte rött "för att det går".

**Det blev ingen konflikt i steg 5.**
Då utgick ni inte från samma commit (någon glömde `git pull` i punkt 1),
eller så ändrade ni olika rader. Kontrollera med `git log -1 --oneline`
hos båda och gör om punkt 1–2 — övningen är byggd så att samma rad + samma
utgångspunkt garanterat krockar.

**Jag mergade fast det var rött/ogranskat — GitHub lät mig.**
Kolla **Enforcement status** på regeln — står den på **Disabled** gäller
den ingen alls, admin eller inte (se diagnosen i steg 2). Står den på
**Active** och det ändå gick igenom: någon har lagt till sig själv i
regelns **Bypass list**. En tom bypass-lista (steg 1) är just det som gör
att skyddet gäller även repo-ägaren — ta bort bypass-entryn. (Och även med
tekniska kryphål: kulturregeln gäller — merga inget ogranskat.)

**Kvarglömda `<<<<<<<`-markörer i filen.**
Markörerna är vanlig text — Git och CI klagar inte, men appen visar dem
för användaren. `git grep '<<<<<<<'` hittar dem; ta bort, committa, pusha
(på en branch + PR, förstås — main är låst, även för städning).

**Min buddy ser inte min PR.**
Skickade du review-förfrågan? PR-sidan → *Reviewers* → välj buddyn. Buddyn
hittar den under **Pull requests** → *Review requests* (klockikonen visar
en avisering).

**Jag taggade `m2-review` innan allt var klart, och committade mer efter.**
Taggen ska peka på committen där milstolpen faktiskt blev klar. Flytta
den — inga force-flaggor, radera och återskapa:

```bash
git tag -d m2-review
git push origin --delete m2-review
git tag m2-review              # på rätt commit
git push origin m2-review
```

Fler tagg- och commit-recept: `slides/referens-git-cheatsheet.md`.
