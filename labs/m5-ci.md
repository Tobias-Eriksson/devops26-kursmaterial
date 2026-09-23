# M5 – Obligatorisk CI-check, röd PR blockeras, egen ändring i ci.yml

Den här labben hör till **session 5** (CI del 1). Ni jobbar i par (eller
solo), ca 45 minuter.

**Milstolpens bevis:** `Lint and test backend` är obligatorisk i
rulesetet för `main`, en medvetet röd pull request har bevisligen
blockerats och sedan mergats grön, `ci.yml` har fått en egen, mergad
ändring (`workflow_dispatch:`), skärmdumparna finns i
`inlamning/m5-ci.md`, och taggen `m5-ci` är uppe.

**Viktigt att veta innan ni börjar:** `.github/workflows/ci.yml` finns
**redan** i ert repo — det följde med från template-repot i M1, och checken
`Lint and test backend` har synts på varje pull request ni öppnat sedan
dess. Den har bara aldrig kunnat **stoppa** något. Dagens lab handlar om
att koppla in den spärren, bevisa att den fungerar, och sedan röra vid
filen själva för första gången.

## Steg 0 – Förkrav

- [ ] Uppdatera kursmaterialet: `cd course-material && git pull`.
- [ ] M1–M3 är klara: taggarna `m1-repo`, `m2-review` och `m3-container`
      är uppe.
- [ ] Rulesetet från M2 är på: **Enforcement status: Active**, **Require
      a pull request before merging**, *Required approvals* **1 om ni är
      ett par, 0 om du jobbar solo**. Inte klar? Gör klart M2 först
      (`labs/m2-review.md`) — dagens lab bygger vidare på exakt samma
      ruleset.
- [ ] Checken `Lint and test backend` var **grön** på er senaste mergade
      pull request (fliken **Checks**, eller sektionen under
      kommentarerna). Syns den inte alls — fråga läraren innan ni går
      vidare. Var den **röd**? Då är `main` redan röd, och steg 3 kan
      aldrig bli grönt — se Vanliga problem först.
- [ ] M4 krävs inte för dagens steg — men gör klart det innan session 10.
- [ ] **Par:** den ena gör steg 2–3 (den röda PR:en), den andra steg 4
      (`workflow_dispatch`), och ni granskar varandras PR — båda ska ha
      en egen PR i M5.
- [ ] **Solo:** *Required approvals* står kvar på **0** — dagens nya
      krav (grön check) gäller dig precis som paren. Du granskar dina
      PR:er som i M2 (beskrivning + minst en egen radkommentar).

## Steg 1 – Gör checken obligatorisk

På GitHub, i ert repo:

1. **Settings** → **Rulesets** → klicka på regeln för `main` (den ni
   skapade i M2).
2. Kryssa i **Require status checks to pass**.
3. **Add checks** (+) → sök upp och välj **Lint and test backend** — det
   är exakt namnet på jobbet i `ci.yml` (`jobs.lint-and-test.name`), inte
   YAML-nyckeln (`lint-and-test`). Välj **bara** den: förslagslistan kan
   också visa `Build and push …` och `SSH deploy …` från körningarna på
   `main` — de kör aldrig på pull requests, och kryssar ni i dem kan ingen
   PR någonsin mergas.
4. Rör inget annat på sidan: *Required approvals* (1 par / 0 solo),
   **Enforcement status: Active** och den tomma bypass-listan står kvar
   från M2.
5. **Save changes**.

**Syns inte checken i sökrutan?** Den måste ha kört minst en gång mot en
pull request vars **bas-branch** är `main` innan GitHub känner till
namnet. Kolla er senaste mergade pull request, bekräfta att checken
faktiskt kört där, och försök igen — eller skriv in namnet exakt för hand.

> 📸 **Kom ihåg skärmdump till inlämningen:** inställningssidan för regeln, med **Require status checks to pass**, `Lint and test backend` i listan och **Enforcement status: Active** synligt.

## Steg 2 – Bevisa att spärren fungerar: en medvetet röd PR

Stå i **ert eget repo**, inte i `course-material` (där steg 0 lämnade er).
I codespacen ligger `course-material` inuti ert repo, som i sin tur ligger
i `/workspaces/` (lokalt: `cd` till er klon):

```bash
cd /workspaces/<ditt-repo>

git switch main && git pull
git switch -c break-the-build
```

Öppna `backend/app/main.py` och gör ett medvetet `ruff`-fel: en oanvänd
import. Placeringen spelar roll — lägg den som en egen rad, med tom rad
på var sida, direkt efter `from __future__ import annotations` och ovanför
`fastapi`/`pydantic`-importerna (annars klagar ruff på fler saker än den
tänkta importen). Filens början ska se ut så här:

```python
from __future__ import annotations

import json  # oanvänd - ruff ska klaga

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
```

```bash
git add backend/app/main.py
git commit -m "test: deliberately break lint to prove branch protection works"
git push -u origin break-the-build
```

Öppna pull requesten. Skriv i beskrivningen varför PR:en finns, och
**"Så testar du"**: `cd backend && ruff check .` ska klaga på `json`
(F401), och checken ska bli röd.

Vänta på att checken kör klart (fliken **Checks** eller uppdatera sidan)
— den ska bli **röd**, och merge-knappen ska vara **grå och oklickbar**
med en text i stil med *"Merging is blocked — Required statuses must
pass"*. Klicka på checken → **Details** och läs loggen: `ruff` ska peka
ut exakt den oanvända importen.

**Granska nu, medan diffen har innehåll** — efter fixen i steg 3 är den
tom. **Par:** buddyn lämnar en radkommentar på `import json`-raden.
**Solo:** du lämnar din radkommentar där (**Files changed** → plusset vid
radnumret → **Review changes** → **Comment**).

> 📸 **Kom ihåg skärmdump till inlämningen:** PR:en med den röda checken och den grå merge-knappen (*Merging is blocked*). Det här är hela poängen med dagens milstolpe — och den går inte att ta i efterhand.

## Steg 3 – Fixa och merga grönt

Ta bort raden ni lade till, committa och pusha:

```bash
git add backend/app/main.py
git commit -m "fix: remove unused import"
git push
```

Checken kör om automatiskt. När den är **grön** och reviewn är gjord blir
merge-knappen klickbar igen. **Par:** buddyn godkänner nu, när checken
är grön (**Review changes** → **Approve**) — egen approve räknas aldrig.
**Solo:** din radkommentar från steg 2 är granskningen.

Sedan, i den här ordningen:

1. **På GitHub — merga PR:en:** **Merge pull request** (standardvalet
   *Create a merge commit*) → **Confirm merge**. PR:en ska nu stå som
   **Merged** (lila).
2. **På GitHub — radera branchen där:** klicka **Delete branch** i
   samma vy. Det tar bort `break-the-build` på GitHub, inte hos er.
3. **Lokalt — hämta mergen och radera er kopia:**

```bash
git switch main && git pull
git branch -d break-the-build
```

Säger git *"not yet merged to HEAD"*? Då saknar er lokala `main` mergen —
oftast för att PR:en inte är mergad än (punkt 1), eller för att `git pull`
kördes före mergen. Kolla att PR:en står som **Merged** och kör `git pull`
igen.

## Steg 4 – Egen ändring: lägg till en manuell trigger

Nu rör ni filen själva för första gången — samma PR-flöde som allt annat
sedan M2. **Par:** nu är det den andras tur.

```bash
git switch main && git pull
git switch -c ci-workflow-dispatch
```

Öppna `.github/workflows/ci.yml` och lägg till `workflow_dispatch:` som
en **extra** rad under `on:` — rör inte `pull_request`-delen:

```yaml
on:
  pull_request:
    branches: [main]
  workflow_dispatch:
```

Indraget avgör allt (samma YAML-regel som föreläsningen): `workflow_dispatch`
ska ligga i **samma kolumn** som `pull_request`, båda under `on:`.

```bash
git add .github/workflows/ci.yml
git commit -m "ci: allow manually triggering the workflow"
git push -u origin ci-workflow-dispatch
```

Öppna pull requesten. Beskrivningen säger varför, och **"Så testar du"**:
efter merge, **Actions** → **CI** → **Run workflow** ska finnas och starta
en körning. Checken kör (samma workflow som alltid — ni ändrade bara
triggern, inte jobbet). **Par:** buddyn granskar och godkänner. **Solo:**
självgranskning som i M2 (minst en radkommentar på er nya rad). Merga,
**Delete branch**.

**Bevisa att triggern fungerar:** när ändringen är mergad till `main`,
gå till fliken **Actions** → workflowen **CI** i vänstermenyn → knappen
**Run workflow** ska nu finnas uppe till höger (den fanns inte innan er
ändring). Välj `main`, klicka **Run workflow**, och se en ny körning
starta **utan** att ni öppnat en pull request.

I Actions-fliken ser ni också `Publish images` och en röd `Deploy to VM`
från er merge — samma väntade röda som i M1 steg 4 (den blir grön i M9).

> 📸 **Kom ihåg skärmdump till inlämningen:** Actions-fliken med er manuellt startade CI-körning (trigger: `workflow_dispatch`).

## Steg 5 – Lämna in beviset och tagga milstolpen

**Bevisa det osynliga:** regeln i Settings, den blockerade merge-knappen
och den manuella körningen syns inte i repot. Ni har nu 3 skärmdumpar från
steg 1, 2 och 4 — lägg dem i `inlamning/` och skriv några meningar per
skärmdump i `inlamning/m5-ci.md` — formatet visas i
`inlamning/m0-exempel.md`. (Skärmdumparna hamnar på er egen dator, inte i
codespacen — dra in filerna i codespacens filutforskare, eller använd
**Upload**.)

`main` är skyddad, så även beviset går in via en PR:

```bash
git switch main && git pull
git switch -c m5-inlamning
git add inlamning/
git commit -m "docs: add M5 proof of the required CI check"
git push -u origin m5-inlamning
```

Öppna PR:en, vänta på grön check, granska (par: buddyn / solo:
radkommentar), merga, **Delete branch**.

**Kontrollera på GitHub** att `workflow_dispatch:` och
`inlamning/m5-ci.md` ligger på `main` — tagga då:

```bash
git switch main && git pull
git tag m5-ci
git push origin m5-ci
```

**AI-verktyg:** samma regel som i M2 — Claude Code, Codex och Gemini i
devcontainern får hjälpa er med YAML-**syntaxen**. Meningarna i
`inlamning/m5-ci.md` skriver ni själva, med egna ord.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] **Require status checks to pass** är på, med `Lint and test backend`
      som enda check (Settings → Rulesets).
- [ ] En medvetet röd pull request har bevisligen blockerat
      merge-knappen (steg 2).
- [ ] Samma ändring är fixad och mergad, checken var grön vid merge
      (steg 3) — granskad av buddyn, eller med radkommentar (solo).
- [ ] `.github/workflows/ci.yml` har `workflow_dispatch:` under `on:`,
      mergad via en egen granskad pull request (steg 4).
- [ ] En manuellt startad körning syns under fliken **Actions**.
- [ ] **I par:** båda har en egen mergad PR i M5.
- [ ] Det osynliga arbetet är dokumenterat med 3 skärmdumpar i
      `inlamning/m5-ci.md`, mergat till `main`.
- [ ] Taggen `m5-ci` pekar på `main` efter sista mergen och syns under
      **Tags**.

## Vanliga problem

**Checken syns inte i listan när jag ska lägga till den i rulesetet
(steg 1).** Workflow-filen (eller ett namnbyte av jobbet) finns bara på en
feature-branch än, inte på `main`. GitHub lär sig bara om checkar som
kört mot pull requests med `main` som **bas**-branch. Merga en vanlig PR
först (eller vänta på att er nästa PR kör klart) och försök igen.

**Checken är röd fast jag tagit bort importen (steg 3).** Då var `main`
redan röd innan ni började — oftast `ruff` som klagar på något från
tidigare milstolpar (en oanvänd import i `test_bugjakt.py`, importer i
fel ordning, en rad över 100 tecken). Läs loggen under **Details**: den
pekar ut fil och rad. Kör samma kontroll i codespacen:
`docker compose run --rm --build backend ruff check .` — fixa det loggen
säger på samma branch, pusha, och checken blir grön.

**Min M4-PR går inte längre att merga.** Rätt — testet i den är rött, och
nu stoppar checken det. Fixa buggen först (`labs/m4-bugjakt.md`); det är
exakt spärren ni just slog på.

**Jag kryssade i en check, men min röda PR blockeras ändå inte.**
Vanligast: **namnbyte**. Om `jobs.<id>.name` i YAML:en någon gång ändrats
utan att rulesetet uppdaterats, pekar regeln på ett gammalt namn som inte
längre existerar — den checken "körs" aldrig, så kravet räknas som
uppfyllt av ingenting. Jämför texten i Settings **exakt**, tecken för
tecken, mot `name:`-raden i `ci.yml`.

**Röd PR blockeras inte, fast namnet stämmer och checken faktiskt är
röd.** Samma diagnos som `labs/m2-review.md` steg 2: antingen är repot
**Private** på ett gratis GitHub-konto (rulesets enforce:as inte alls
där — byt till Public, eller gå med i GitHub Education), eller så står
regelns **Enforcement status** kvar på **Disabled**, eller så har någon
lagt in sig själv i **Bypass list** (den ska vara tom). Fixa enligt M2
steg 2 och testa om.

**Min PR kan inte mergas trots grön check (Required approvals).** Samma
regel som alltid sedan M2: är ni ett par kräver rulesetet en godkänd
review — be er buddy titta. Solo: *Required approvals* är 0, så
självgranska PR:en som i M2 och merga. Står den på 1 för dig som jobbar
solo — sätt tillbaka den till 0 (M2 steg 1).

**Min körning står och väntar ("Queued") i flera minuter.** Inget fel —
GitHubs delade, gratis runners är en delad kö som kan bli upptagen vid
hög belastning (t.ex. hela klassen pushar samtidigt). Vänta, den startar.

**`workflow_dispatch`-knappen syns inte i Actions-fliken efter min
merge.** Kontrollera indraget: `workflow_dispatch:` måste ligga i SAMMA
kolumn som `pull_request:`, båda direkt under `on:`. Fel indrag gör
antingen att YAML:en blir ogiltig (checken körs aldrig alls, syns som
fel i Actions) eller att nyckeln hamnar på fel plats i strukturen.

**Taggen pekar på fel commit / syns inte på GitHub.** Samma recept som
M1:s Vanliga problem: `git tag -d m5-ci`, `git push --delete origin
m5-ci`, `git switch main && git pull`, tagga om, `git push origin m5-ci`.
