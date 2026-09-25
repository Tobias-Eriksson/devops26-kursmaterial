# M6 – Egen ändring i publish-images.yml, publika paket, pull, secrets-övning

Den här labben hör till **session 6** (CI del 2). Ni jobbar i par (eller
solo), ca 60 minuter.

**Milstolpens bevis:** en egen, mergad ändring i `publish-images.yml`
har bevisligen triggat ett automatiskt bygge+push vid merge till `main`,
båda GHCR-paketen är publika, en image är pullad utan inloggning,
`GITHUB_TOKEN` har setts maskerat i en logg (och debug-raden är
borttagen igen), skärmdumparna finns i `inlamning/m6-cd-images.md`, och
taggen `m6-cd-images` är uppe.

**Viktigt att veta innan ni börjar:** `.github/workflows/publish-images.yml`
finns **redan** i ert repo — det följde med från template-repot i M1,
precis som `ci.yml` gjorde inför M5. Det har körts automatiskt vid varje
merge till `main` sedan dess (ni såg den gröna körningen redan i M1
steg 4), men ingen av er har rört filen eller följt kedjan hela vägen
till en pullbar image. Dagens lab handlar om att ändra i den själva för
första gången, och sedan följa hela kedjan från merge till pullbar image.

## Steg 0 – Förkrav

- [ ] Uppdatera kursmaterialet: `cd course-material && git pull`.
- [ ] M1–M3 och M5 är klara: taggarna `m1-repo`, `m2-review`,
      `m3-container` och `m5-ci` är uppe. I M3 loggade ni in i GHCR med
      en PAT och pushade båda images för hand (`labs/m3-container.md`).
- [ ] Rulesetet från M2 + M5 är på: **Enforcement status: Active**,
      **Require a pull request before merging**, *Required approvals*
      **1 om ni är ett par, 0 om du jobbar solo**, och **Require status
      checks to pass** med `Lint and test backend`. Dagens PR:er går
      igenom exakt samma spärr.
- [ ] Checken `Lint and test backend` var **grön** på er senaste mergade
      pull request. Var den **röd**? Då är `main` redan röd och ingen PR
      i dag kan mergas — se Vanliga problem i `labs/m5-ci.md` först.
- [ ] M4 krävs inte för dagens steg — men gör klart det innan session 10.
- [ ] **Par:** den ena gör steg 2 (PR:en i `publish-images.yml`), den
      andra steg 6–7 (secrets-övningen och inlämnings-PR:en); steg 3–5
      gör ni ihop, och ni granskar varandras PR — båda ska ha en egen
      mergad PR i M6.
- [ ] **Solo:** *Required approvals* står kvar på **0** — dagens krav
      (grön check) gäller dig precis som paren. Du granskar dina PR:er
      som i M2 och M5 (beskrivning + minst en egen radkommentar).

## Steg 1 – Läs `publish-images.yml` och jämför med `ci.yml`

Öppna `.github/workflows/publish-images.yml`. Jämför mot `ci.yml`
(M5) rad för rad:

1. `on: push` mot `branches: [main]` — inte `pull_request`. Varför körs
   den bara vid merge, inte vid varje uppdatering av en öppen PR?
   (Svar: ni vill bara bygga och publicera det som faktiskt är godkänt
   och landat, inte varje utkast.)
2. `permissions: contents: read, packages: write` — jämför med `ci.yml`,
   som inte har något eget `permissions`-block alls. Varför behöver just
   den här workflowen extra rättigheter? (Svar: den ska pusha till ett
   paket-register, `ci.yml` gör bara `pip install` + kör tester.)
3. Hitta `docker/login-action@v3` — vilka två värden loggar den in med,
   och varifrån kommer de? (`github.actor` + `secrets.GITHUB_TOKEN`,
   båda från körmiljön, ingenting en människa skapat.)
4. Hitta steget **Lowercase repository owner** (`id: owner`). Det är
   M3:s gemener-fälla (`repository name must be lowercase`) löst en
   gång för alla: `${GITHUB_REPOSITORY_OWNER,,}` gör ägarnamnet till små
   bokstäver, och stegen efter läser `steps.owner.outputs.owner` — det
   använder ni själva i steg 2.
5. Hitta taggarna i `docker/build-push-action@v5`: en `:latest` och en
   `:${{ github.sha }}`. Vilken av de två skulle ni lita på om ni skulle
   återskapa exakt den image som kördes för tre veckor sedan?

## Steg 2 – Egen ändring: manuell trigger + spårbarhetssteg

Nu rör ni filen själva för första gången — samma PR-flöde som allt annat
sedan M2. Stå i **ert eget repo**, inte i `course-material` (där steg 0
lämnade er). I codespacen ligger `course-material` inuti ert repo, som i
sin tur ligger i `/workspaces/` (lokalt: `cd` till er klon):

```bash
cd /workspaces/<ditt-repo>

git switch main && git pull
git switch -c publish-images-workflow-dispatch
```

Öppna `.github/workflows/publish-images.yml`. Lägg till
`workflow_dispatch:` som en **extra** rad under `on:` (rör inte
`push`-delen):

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

Indraget avgör allt (samma regel som i M5): `workflow_dispatch` ska ligga
i **samma kolumn** som `push`, båda under `on:`.

Lägg sedan till ett nytt sista steg i **båda** jobben
(`build-and-push-backend` och `build-and-push-frontend`), direkt efter
`Build and push`-steget, som skriver bygglets taggar till
körningens sammanfattning:

```yaml
      - name: Record image tags
        run: |
          {
            echo "### Taggar för det här bygget"
            echo "- \`ghcr.io/${{ steps.owner.outputs.owner }}/template-app-backend:latest\`"
            echo "- \`ghcr.io/${{ steps.owner.outputs.owner }}/template-app-backend:${{ github.sha }}\`"
          } >> "$GITHUB_STEP_SUMMARY"
```

(Byt `template-app-backend` mot `template-app-frontend` i
frontend-jobbets steg.) Det här är avsiktligt en **tilläggande**
ändring — den kan inte förstöra det befintliga bygget eller pushen,
den bara lägger till ett extra, läsbart bevis i Actions-gränssnittet.
Rör inget annat i filen — särskilt inte `push:`-raden i `Build and push`
eller steget **Lowercase repository owner** (Vanliga problem förklarar
varför).

```bash
git add .github/workflows/publish-images.yml
git commit -m "ci: add manual trigger and tag summary to publish-images"
git push -u origin publish-images-workflow-dispatch
```

Öppna pull requesten. Beskrivningen säger varför, och **"Så testar du"**:
diffen är två rena tillägg (inget borttaget), och efter merge ska
**Actions** → **Publish images** visa en körning startad av mergen vars
sammanfattning listar två taggar per jobb, och knappen **Run workflow**
ska finnas. Checken `Lint and test backend` kör som vanligt (ni rörde
inte `ci.yml`, den ska vara grön). **Par:** buddyn granskar de nya
raderna och godkänner (**Review changes** → **Approve**) — egen approve
räknas aldrig. **Solo:** självgranskning som i M2 (minst en radkommentar
på `workflow_dispatch:`-raden). Merga, **Delete branch**.

## Steg 3 – Verifiera automatiken end-to-end

Er merge i steg 2 är själva push-till-main-händelsen — den ska trigga
`publish-images.yml` automatiskt, precis som varje merge gjort sedan M1.

1. Gå till fliken **Actions** → workflowen **Publish images** →
   senaste körningen. Den ska ha startat automatiskt av er merge-commit
   (inte av er, manuellt).
2. Öppna körningen, kolla att **båda** jobben (**Build and push backend
   image**, **Build and push frontend image**) är gröna.
3. Stanna på körningens översikt (**Summary** överst i vänstermenyn) och
   scrolla ner under jobbgrafen — där finns ett avsnitt per jobb med
   rubriken *Taggar för det här bygget* och exakt de två taggar som
   pushades. Det är spårbarhetssteget ni la till.

I Actions-fliken ser ni också en röd `Deploy to VM` efter varje
`Publish images`-körning på `main` — samma väntade röda som i M1 steg 4
och M5 (den blir grön i M9). Bara `Publish images` ska vara grön i dag.

> 📸 **Kom ihåg skärmdump till inlämningen:** körningens Summary-sida — båda jobben gröna och avsnittet *Taggar för det här bygget* synligt.

**Bevisa att `workflow_dispatch` fungerar också:** knappen **Run
workflow** ska nu finnas uppe till höger på workflow-sidan (fanns inte
innan er ändring). Kör den manuellt en gång, välj `main`, se en ny
körning starta utan att ni öppnat en PR.

> 📸 **Kom ihåg skärmdump till inlämningen:** listan över körningar för **Publish images** med er manuellt startade körning (trigger: `workflow_dispatch`).

## Steg 4 – Gör paketen publika

Precis som M3:s Vanliga problem och README:s "Paket-synlighet"-avsnitt
varnar för: paketen skapades av CI redan vid M1 och är **privata** som
standard. **Par:** repots ägare gör det här steget — bara ägaren kan
ändra paketens synlighet.

1. Gå till **Packages** i repots högerspalt på GitHub (eller fliken
   **Packages** på er profil).
2. Öppna `template-app-backend` → **Package settings** → **Change
   visibility** → **Public**, skriv paketets namn i rutan och bekräfta.
3. Upprepa för `template-app-frontend`.

Står ett paket redan på **Public** (ni kanske gjorde det redan i M3)? Då
är steget klart för det paketet — ta skärmdumpen ändå.

> 📸 **Kom ihåg skärmdump till inlämningen:** paketsidan för ett av paketen med **Public** synligt.

## Steg 5 – Pull en image utan inloggning

Logga **ut** ur GHCR först, så ni verkligen testar den publika
åtkomsten och inte bara återanvänder M3:s inloggning. Kommandona nedan
är desamma i codespacen som på en egen dator; svarar `docker logout`
att ni inte var inloggade är det inget fel — då testar ni redan
anonym åtkomst. Byt ut
`<ert-användarnamn>` mot ert GitHub-användarnamn **med små bokstäver** —
Docker kräver gemener i sökvägen, även om användarnamnet på GitHub har
versaler:

```bash
docker logout ghcr.io
docker pull ghcr.io/<ert-användarnamn>/template-app-backend:latest
```

Pullen ska lyckas **utan** att be om inloggning. Kör den för att
bekräfta att det verkligen är samma bygge — `-d` startar containern i
bakgrunden och `sleep 3` ger den tid att starta:

```bash
docker run -d --name m6-pull-test -p 8000:8000 ghcr.io/<ert-användarnamn>/template-app-backend:latest
sleep 3
curl http://localhost:8000/api/health
docker rm -f m6-pull-test
```

Svaret ska vara `{"status":"ok"}`.

> 📸 **Kom ihåg skärmdump till inlämningen:** terminalen med `docker logout`, `docker pull` som lyckas utan inloggning och `curl`-svaret från `/api/health`.

## Steg 6 – Secrets-övning: se maskeringen på riktigt

**Läs det här helt innan ni börjar.** Övningen är säker av tre skäl,
och alla måste stämma för att det ska vara okej:

- `GITHUB_TOKEN` **maskeras** automatiskt av GitHub Actions — värdet
  ersätts med `***` i loggen, oavsett var det dyker upp.
- Även om maskeringen skulle missa något är `GITHUB_TOKEN` **redan
  värdelös** en minut senare — den existerar bara under själva
  jobbkörningen.
- Körningen **pushar ingenting** — `push:` i `Build and push` är sant
  bara på `main`, så `latest` och sha-taggarna i GHCR rör sig inte
  (samma guard som genomgången på föreläsningen).

**Reflexen att prova samma sak med en riktig, långlivad secret (en PAT,
en molnnyckel) är farlig.** Maskering är ett skyddsnät mot misstag i
loggen, inte en garanti — och en long-lived secret som läcker fortsätter
fungera långt efter att ni glömt bort testet. Gör den här övningen
**bara** med `GITHUB_TOKEN`, aldrig med en egen token.

**Kör övningen** på en egen, ogranskad branch — **öppna aldrig en PR
för den här ändringen**:

```bash
git switch main && git pull
git switch -c secrets-masking-demo
```

Lägg till en temporär rad längst ner i `build-and-push-backend`-jobbet:

```yaml
      - name: TEMP - do not merge
        run: echo "${{ secrets.GITHUB_TOKEN }}"
```

```bash
git add .github/workflows/publish-images.yml
git commit -m "test: temporary secret echo for masking demo (never merge)"
git push -u origin secrets-masking-demo
```

Gå till **Actions** → **Publish images** → **Run workflow**, välj er
branch `secrets-masking-demo`, kör. Öppna loggen för steget **TEMP - do
not merge** — värdet ska visas som `***`, aldrig i klartext.

Körningen bygger båda images och visar maskeringen, men pushar
ingenting — bara `main` får publicera. (Spårbarhetssteget skriver ändå
sin sammanfattning — taggarna som listas där är de bygget SKULLE ha
fått, inte taggar som faktiskt publicerades.)

> 📸 **Kom ihåg skärmdump till inlämningen:** loggen för steget **TEMP - do not merge** med `***` i stället för tokenvärdet.

**Städa upp direkt efteråt — detta får aldrig mergas:**

```bash
git switch main
git branch -D secrets-masking-demo
git push origin --delete secrets-masking-demo
```

## Steg 7 – Lämna in beviset och tagga milstolpen

**Bevisa det osynliga:** Actions-körningarna, synlighetsändringen till
**Public**, `docker pull` utan inloggning och den maskerade secreten
(`***`) syns inte i repot. Ni har nu 5 skärmdumpar från steg 3, 4, 5
och 6 — lägg dem i `inlamning/` och skriv några meningar per skärmdump
i `inlamning/m6-cd-images.md` — formatet visas i
`inlamning/m0-exempel.md`. (Skärmdumparna hamnar på er egen dator, inte i
codespacen — dra in filerna i codespacens filutforskare, eller använd
**Upload**.)

`main` är skyddad sedan M2, så även beviset går in via en PR:

```bash
git switch main && git pull
git switch -c m6-inlamning
git add inlamning/
git commit -m "docs: add M6 proof of the automated image publish"
git push -u origin m6-inlamning
```

Öppna PR:en, vänta på grön check, granska (par: buddyn / solo:
radkommentar), merga, **Delete branch**.

**Kontrollera på GitHub** att `workflow_dispatch:`, steget `Record image
tags` och `inlamning/m6-cd-images.md` ligger på `main` — tagga då:

```bash
git switch main && git pull
git tag m6-cd-images
git push origin m6-cd-images
```

**AI-verktyg:** samma regel som i M2 och M5 — Claude Code, Codex och
Gemini i devcontainern får hjälpa er med YAML-**syntaxen**. Meningarna i
`inlamning/m6-cd-images.md` — särskilt varför övningen i steg 6 är säker
med `GITHUB_TOKEN` men inte med en PAT — skriver ni själva, med egna ord.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] `publish-images.yml` har `workflow_dispatch:` och ett
      spårbarhetssteg som skriver till `$GITHUB_STEP_SUMMARY`, mergat
      via en egen granskad pull request (steg 2) — granskad av buddyn,
      eller med radkommentar (solo).
- [ ] En merge till `main` triggade `Publish images` automatiskt, båda
      jobben gröna, sammanfattningen visar de publicerade taggarna.
- [ ] En manuellt startad körning (`workflow_dispatch`) syns under
      **Actions**.
- [ ] Båda GHCR-paketen (`-backend`, `-frontend`) är **Public**.
- [ ] `docker pull` av backend-imagen lyckades **utan** inloggning, och
      containern svarade på `/api/health`.
- [ ] `GITHUB_TOKEN` sågs maskerat (`***`) i en logg, och
      `secrets-masking-demo`-branchen är **raderad**, aldrig mergad.
- [ ] **I par:** båda har en egen mergad PR i M6.
- [ ] Det osynliga arbetet är dokumenterat med 5 skärmdumpar i
      `inlamning/m6-cd-images.md`, mergat till `main`.
- [ ] Taggen `m6-cd-images` pekar på `main` efter sista mergen och syns
      under **Tags**.

## Vanliga problem

**Checken `Lint and test backend` är röd på min PR fast jag bara rört
`publish-images.yml`.** Då var `main` redan röd innan ni började — samma
diagnos som M5:s Vanliga problem ("Checken är röd fast jag tagit bort
importen"): läs loggen under **Details**, fixa det den pekar på i samma
PR, pusha.

**Min PR kan inte mergas trots grön check (Required approvals).** Samma
regel som sedan M2: är ni ett par kräver rulesetet en godkänd review —
be er buddy titta. Solo: *Required approvals* är 0, så självgranska
PR:en som i M2 och merga. Står den på 1 för dig som jobbar solo — sätt
tillbaka den till 0 (M2 steg 1).

**Körningen står och väntar ("Queued") eller bygger i flera minuter.**
Inget fel — `Publish images` bygger två images på GitHubs delade, gratis
runners; vid hög belastning (hela klassen mergar samtidigt) blir kön
lång. Vänta.

**`docker pull` svarar `denied: requested access to the resource is
denied`.** Paketet är fortfarande privat — gå tillbaka till steg 4,
**Package settings → Change visibility → Public**. Samma fälla som
nämndes redan i M3, nu med skarpt läge.

**`docker pull` svarar `invalid reference format: repository name must be
lowercase`.** Ert GitHub-användarnamn har versaler och ni skrev in det
ordagrant i `docker pull`/`docker run`-kommandot (steg 5). Docker tillåter
bara gemener i repository-sökvägen — skriv om `<ert-användarnamn>` med små
bokstäver och kör om kommandot. Samma regel som mötte er redan vid
`docker push` i M3.

**`docker run` i steg 5 svarar `bind: address already in use`.** Något
lyssnar redan på port 8000 — oftast `docker compose up` från M3 eller en
kvarglömd container. `docker ps`, stoppa den, kör om.

**`curl` svarar `Connection refused` i steg 5.** Containern hade inte
hunnit starta — vänta någon sekund och kör `curl` igen. Svarar den
fortfarande inte: `docker logs m6-pull-test`.

**`docker pull` svarar `no matching manifest for linux/arm64/v8`** (Mac
med M-chip, lokalt — codespacen påverkas inte). CI byggde imagen bara
för amd64. Lägg till `--platform linux/amd64` på både `docker pull` och
`docker run`, så kör Docker den emulerat och containern svarar som
vanligt.

**Pushen i workflowen failar, eller taggen ser konstig ut
(dubbla understreck, stora bokstäver kvar).** GHCR kräver **gemener**
i hela sökvägen. Workflowen har redan ett steg som gemenererar
`github.repository_owner` — kontrollera att ni inte råkat ta bort eller
skriva om det steget när ni lade till spårbarhetssteget i steg 2.

**Pushen failar med `denied`/403 trots att inloggningen lyckas.**
`permissions: packages: write` saknas eller har tagits bort från
workflow-filens topp. Ett vanligt misstag om man experimenterar och
råkar radera raden — jämför mot originalet.

**Jag ser inte `Run workflow`-knappen i Actions.** `workflow_dispatch:`
måste ligga i SAMMA kolumn som `push:`, direkt under `on:` — samma
indrags-fälla som i M5. Vänta också tills PR:en i steg 2 faktiskt är
mergad till `main`, knappen dyker bara upp för workflower som finns där.

**Jag glömde radera `secrets-masking-demo`-branchen.** Gör det nu:
`git push origin --delete secrets-masking-demo`. Kontrollera samtidigt
att debug-raden aldrig kom med i en PR mot `main` — om den gjorde det,
öppna en ny PR som tar bort raden och merga den innan ni går vidare.

**Taggen pekar på fel commit / syns inte på GitHub.** Samma recept som
M1:s Vanliga problem: `git tag -d m6-cd-images`, `git push --delete origin
m6-cd-images`, `git switch main && git pull`, tagga om, `git push origin
m6-cd-images`.
