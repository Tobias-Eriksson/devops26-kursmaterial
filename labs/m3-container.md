# M3 – Dockerfiles, docker compose och push till GHCR

Den här labben hör till **session 3** (Containers).

**Milstolpens bevis:** appen kör med `docker compose` i er arbetsmiljö
(codespacen eller lokalt), båda images (`template-app-backend`,
`template-app-frontend`) finns i GHCR, en egen
`.dockerignore`-ändring är mergad via granskad PR, och taggen
`m3-container` är uppe.

**Viktigt att veta innan ni börjar:** `backend/Dockerfile`,
`frontend/Dockerfile` och `docker-compose.yml` finns **redan** i ert repo —
de följde med från template-repot i M1. Dagens lab handlar inte om att
skriva dem från noll, utan om att **läsa dem tillräckligt noga för att
förstå varje rad**, bevisa den förståelsen genom att förutsäga vad som
händer innan ni kör, och sedan lägga till något **eget**.

## Steg 0 – Förkrav

- [ ] Uppdatera kursmaterialet: `cd course-material && git pull`.
- [ ] M1 är klar: repot finns och taggen `m1-repo` är uppe.
- [ ] `docker version` svarar utan fel. **I codespacen är Docker redan på
      plats** (devcontainern kör docker-in-docker) — kör ni lokalt
      betyder det att Docker Desktop eller motsvarande är igång.
- [ ] M2 är klar: branch protection är på och taggen `m2-review` är uppe.
      Dagens egna ändring (steg 4) ska gå via branch + PR + review — exakt
      det ni övade i M2, och det är så den bedöms.

**Kör ni lokalt och Docker krånglar?** Det händer — disk full, Docker
Desktop startar inte, WSL2-problem på Windows. Byt till en **Codespace**:
på repots GitHub-sida, **Code** → **Codespaces** → **Create codespace on
main** (jobbar ni redan i en codespace berör det här er inte). Devcontainern
har Docker inbyggt (docker-in-docker) och portarna 8080/8000
vidarebefordras automatiskt — alla kommandon i den här labben fungerar
identiskt i codespacen. Säg gärna till läraren så ni slipper felsöka en
trasig lokal Docker-installation mitt i labben.

**Så här öppnar ni en vidarebefordrad port i Codespacen** (den här labbens
`localhost:8080`/`localhost:8000`-instruktioner pekar hit): panelen
bredvid **Terminal** har en flik **Ports** — hitta raden för porten (t.ex.
`8080`), klicka glob-ikonen **Open in Browser**. Adressen ser ut som
`https://<codespace-namn>-8080.app.github.dev`, inte `localhost`.
Alternativt: klicka på toasten "Your application running on port 8080 is
available" som dyker upp när tjänsten startar, eller Cmd/Ctrl-klicka på en
`localhost`-länk direkt i terminalen — VS Code skriver om den åt er.
Forwarden är **privat** som standard: den kräver inloggning med samma
GitHub-konto som äger codespacen.

## Steg 1 – Läs och förutsäg: backend/Dockerfile

Öppna `backend/Dockerfile`. Läs den rad för rad och **skriv ner era
gissningar (inlamning/m3-container.md)!** innan ni läser vidare eller kör något:

1. Vilken rad avgör Python-versionen imagen bygger på?
2. `COPY requirements.txt .` och `RUN pip install ...` kommer FÖRE `COPY
   app ./app`. Varför i den ordningen, och inte tvärtom? (Ledtråd:
   föreläsningens cache-slide.)
3. `EXPOSE 8000` — tror ni den raden gör porten nåbar utanför containern,
   så att `curl` i er terminal når den? Testa gissningen i nästa
   deluppgift.

Raden `RUN apt-get update && apt-get -y upgrade ...` högst upp är
OS-patchning av basimagen — den hör till säkerhetsspåret (Spår C) och ni kan
hoppa över den idag.

**Testa gissning 3 på riktigt:**

```bash
docker build -t backend-test ./backend
docker run -d --name expose-test backend-test
curl http://localhost:8000/api/health   # vad händer?
```

Anropet ska **misslyckas** (connection refused) — `EXPOSE` är bara
dokumentation, den publicerar ingen port. Nu med porten faktiskt kopplad:

```bash
docker rm -f expose-test
docker run -d --name expose-test -p 8000:8000 backend-test
curl http://localhost:8000/api/health   # nu ska den svara {"status":"ok"}
docker rm -f expose-test
```

`-p 8000:8000` (eller `ports:` i compose, se steg 3) är det som faktiskt
kopplar en port på värden ni kör Docker på — er codespace eller er egen
dator — till containern. `EXPOSE` är bara en läsbarhets-kommentar i
Dockerfilen.

> 📸 **Kom ihåg skärmdump till inlämningen:** terminalen med båda `curl`-anropen — det som misslyckades utan `-p` och det som svarade `{"status":"ok"}` med.

## Steg 2 – Läs frontend/Dockerfile

Öppna `frontend/Dockerfile` — mycket kortare. Fundera:

- Varför finns ingen `RUN`-rad här, till skillnad från backend?
- Varför inget eget `CMD`? (Basimagen `nginxinc/nginx-unprivileged:alpine`
  har redan ett.)
- `nginx.conf` kopieras in som webbserverns konfiguration — öppna filen,
  hitta raden som pratar med `backend:8000`. Var kommer namnet `backend`
  ifrån? (Svar i steg 3 — det är inget magiskt, det är compose-filens
  tjänstenamn.)
- Varför `nginx-unprivileged` och inte vanliga `nginx`? Standard-imagen
  kör som `root` inuti containern och lyssnar på port 80. Det fungerar i
  er utvecklingsmiljö (codespace eller lokalt) men kraschar direkt på
  många plattformar i drift — vår egen
  frontend dog med `mkdir() /var/cache/nginx/... failed (13: Permission
  denied)` när den kördes på Rahti, som av säkerhetsskäl startar varje
  container som en slumpmässig icke-root-användare. `nginx-unprivileged`
  är byggt för att köras som vem som helst och lyssnar därför på 8080
  (portar under 1024 kräver root). Välj basimage efter var koden ska
  köra, inte bara efter vad som råkar funka lokalt.

## Steg 3 – Kör hela appen med docker compose

Öppna `docker-compose.yml` i repo-roten och läs den som en YAML-fil (kartor
och listor, precis som på föreläsningen):

```bash
docker compose up --build
```

Öppna <http://localhost:8080> — appen ska svara. (I Codespace: se Steg 0
för hur ni når den vidarebefordrade porten — `localhost:8080` fungerar
inte direkt i webbläsaren där.) Testa API:t direkt också:

```bash
curl http://localhost:8080/api/health
curl http://localhost:8080/api/items
```

`frontend`:s nginx proxar `/api/...`-anrop vidare till `backend:8000` — det
fungerar eftersom Compose ger varje tjänst sitt **eget tjänstenamn** som
DNS-namn på ett internt nätverk. Det är svaret på steg 2:s fråga.

> 📸 **Kom ihåg skärmdump till inlämningen:** appen i webbläsaren på port 8080 (i Codespace: den vidarebefordrade adressen) och terminalen där `docker compose up` visar båda tjänsterna igång.

Stäng ner när ni är klara att gå vidare (images behålls, bara containers tas bort):

```bash
docker compose down
```

## Steg 4 – Egen utökning: .dockerignore

Nu er egen, riktiga Dockerfile-relaterade ändring — inte en kopia av något
som redan finns.

**Varför det behövs:** även om `COPY app ./app` bara kopierar namngivna
mappar, skickar Docker **hela build-kontexten** (allt i katalogen ni pekar
`docker build` på) till daemonen innan filtreringen sker. Kör ni `pytest`
eller `ruff` lokalt (utanför Docker) skapar de `.pytest_cache/`- och
`.ruff_cache/`-mappar i `backend/` som skickas med i kontexten helt i
onödan — och `__pycache__/`-mapparna under `app/` och `tests/` hamnar
dessutom **i imagen** via `COPY app ./app` / `COPY tests ./tests`.

**Bevisa det själva:**

```bash
mkdir -p backend/app/__pycache__
echo "test" > backend/app/__pycache__/fake.pyc
docker build -t backend-nocheck ./backend
docker run --rm backend-nocheck find /app -iname "*pycache*"
```

Ni ska se `/app/app/__pycache__` i utskriften — den skräpfilen ligger nu i
er image. Städa bort testfilen: `rm -rf backend/app/__pycache__`.

**Fixa det på riktigt** — på en egen branch (M2:s arbetssätt gäller
fortfarande: allt via PR):

**Rekommendation för par:** dela upp arbetet — en tar
`backend/.dockerignore`, den andra `frontend/.dockerignore`, och ni
granskar varandras PR:ar. Då får båda både skriva en PR och granska en,
och git-historiken visar vem som gjort vad — bra underlag vid den
individuella bedömningen. **Solo:** gör båda filerna själv och
självgranska PR:en som i M2 (`labs/m2-review.md`, steg 4).

Kommandona nedan visar solo-flödet med en branch för båda filerna. **Par:**
gör var sin branch (t.ex. `add-backend-dockerignore` och
`add-frontend-dockerignore`), lägg bara till er egen fil, och granska
varandras PR:ar.

```bash
git switch main && git pull
git switch -c add-dockerignore
```

Skapa `backend/.dockerignore`:

```text
**/__pycache__/
**/*.pyc
**/.pytest_cache/
**/.ruff_cache/
```

Och `frontend/.dockerignore` (mindre dramatiskt eftersom frontend-Dockerfilen
redan kopierar namngivna filer, men bra hygien mot build-kontexten):

```text
**/.DS_Store
**/Thumbs.db
Dockerfile
.dockerignore
```

`Thumbs.db` är Windows motsvarighet till macOS `.DS_Store`. `Dockerfile`
och `.dockerignore` ligger också i kontexten fast ingen `COPY`-rad någonsin
behöver dem — receptet behöver inte skickas med råvarorna.

**Observera mönstret:** `__pycache__/` utan `**/`-prefix hade bara
matchat en mapp direkt i `backend/` — INTE `backend/app/__pycache__` som
ligger en nivå ner. Docker läser `.dockerignore` annorlunda än
`.gitignore`: `**/` behövs för att matcha "var som helst i trädet". Testa
båda varianterna om ni är nyfikna — det är den snabbaste vägen att verkligen
förstå skillnaden.

Bevisa att den fungerar (samma test som nyss, nu med filen på plats):

```bash
mkdir -p backend/app/__pycache__
echo "test" > backend/app/__pycache__/fake.pyc
docker build --no-cache -t backend-check ./backend
docker run --rm backend-check find /app -iname "*pycache*"   # ska vara TOM
rm -rf backend/app/__pycache__
```

> 📸 **Kom ihåg skärmdump till inlämningen:** terminalen med båda `find`-körningarna — `/app/app/__pycache__` i utskriften före `.dockerignore`, tom utskrift efter.

Frontend-Dockerfilen kopierar bara namngivna filer, så där kan ingen
skräpfil hamna i imagen.

Committa, pusha, öppna PR, be om review, merga (delete branch) — precis
som M2:

```bash
git add backend/.dockerignore frontend/.dockerignore
git commit -m "Add .dockerignore to keep build context clean"
git push -u origin add-dockerignore
```

## Steg 5 – Logga in och pusha till GHCR

**Skapa en token** (en gång): GitHub → er profilbild → **Settings** →
**Developer settings** → **Personal access tokens** → **Tokens (classic)**
→ **Generate new token** → **Generate new token (classic)**. 

Namnge den, ändra expire date till 90 dagar. Kryssa i scopet **`write:packages`** (som även
ger `read:packages`). 
**Generate token** → Kopiera token — den visas bara en gång.

**Logga in:**

```bash
export CR_PAT=<er-token>
echo $CR_PAT | docker login ghcr.io -u <ert-användarnamn> --password-stdin
```

**Tagga och pusha båda images** (bygg om först om ni gjort ändringar). Byt ut
`<ert-användarnamn>` mot ert GitHub-användarnamn **med små bokstäver** —
Docker kräver gemener i sökvägen, även om användarnamnet på GitHub har
versaler:

```bash
docker compose build

docker tag template-app-backend:latest ghcr.io/<ert-användarnamn>/template-app-backend:latest
docker push ghcr.io/<ert-användarnamn>/template-app-backend:latest

docker tag template-app-frontend:latest ghcr.io/<ert-användarnamn>/template-app-frontend:latest
docker push ghcr.io/<ert-användarnamn>/template-app-frontend:latest
```

**Kontrollera:** repots **Packages**-flik på GitHub (eller er profils) —
paketen fanns redan där (workflown `publish-images.yml` har pushat dem
vid varje merge till `main` sedan M1), så leta efter BEVISET på er egen
push: er `:latest`-tagg med en färsk tidsstämpel under respektive pakets
**Versions**-flik. Det här gjorde ni **för hand** idag — i M6 öppnar ni
den workflown själva och äger den.

> 📸 **Kom ihåg skärmdump till inlämningen:** **Versions**-fliken för ett av paketen, med er egen `:latest`-push och dess färska tidsstämpel synlig.

## Steg 6 – Tagga milstolpen

```bash
git switch main && git pull
git tag m3-container
git push origin m3-container
```

**Bevisa det osynliga:** er `docker compose up`-körning och era
paket på GHCR syns inte i repot. Ni har nu 4 skärmdumpar från steg 1, 3,
4 och 5 — lägg dem i `inlamning/` och skriv några meningar per skärmdump
i `inlamning/m3-container.md`, tillsammans med gissningarna från steg 1,
och committa — formatet visas i `inlamning/m0-exempel.md`.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] `docker compose up --build` startar båda tjänsterna utan fel.
- [ ] Appen svarar på <http://localhost:8080> (i Codespace: via den
      vidarebefordrade porten, se Steg 0), och `/api/health` +
      `/api/items` svarar via proxyn.
- [ ] `backend/.dockerignore` och `frontend/.dockerignore` finns, mergade
      via en granskad PR (inte pushade direkt till `main`).
- [ ] Er egen `:latest`-push syns med färsk tidsstämpel under **Versions**
      för båda paketen under **Packages** på GitHub.
- [ ] Taggen `m3-container` syns under **Tags**.
- [ ] Det osynliga arbetet är dokumenterat i `inlamning/m3-container.md`.

## Vanliga problem

**`bind: address already in use` när jag kör `docker compose up`.**
Något annat program i er miljö (codespace eller egen dator) lyssnar redan
på port 8080 (eller 8000).
Ändra HOST-sidan av portmappningen i `docker-compose.yml`, t.ex.
`"8081:8080"` — container-sidan (`8080`) ska vara oförändrad, den styrs av
vad servern lyssnar på inuti containern (`listen 8080;` i `nginx.conf`).

**`docker build` tar lika lång tid varje gång, trots att jag bara ändrat
`app/main.py`.** Kontrollera ordningen i `backend/Dockerfile`: `COPY
requirements.txt .` + `RUN pip install` måste stå FÖRE `COPY app ./app`.
Står de i fel ordning ogiltigförklarar varje kodändring även
paketinstallationen — dagens cache-fälla, på riktigt.

**`docker push` svarar `denied: permission_denied` eller `unauthorized`.**
Er token saknar `write:packages`-scopet, eller `docker login` gjordes med
fel användarnamn. Skapa om token med rätt scope (steg 5) och logga in på
nytt.

**`docker push` svarar `invalid reference format: repository name must be
lowercase`.** Ert GitHub-användarnamn har versaler och ni skrev in det
ordagrant i `docker tag`/`docker push`-kommandot (steg 5). Docker tillåter
bara gemener i repository-sökvägen — skriv om `<ert-användarnamn>` med små
bokstäver och kör om `docker tag` + `docker push`. Samma fälla möter
`publish-images.yml` i CI, därför har workflown ett eget steg som gör exakt
den omvandlingen (`${GITHUB_REPOSITORY_OWNER,,}`).

**Paketet är pushat men går inte att `pull` utan inloggning.** Paketen
skapades av CI (`publish-images.yml`) redan vid M1 och är **privata som
standard** tills ni ändrar dem. Gå till paketets sida
(**Packages** på repot eller er profil) → **Package settings** → **Change
visibility** → **Public**. Det spelar ingen roll för M3 i sig, men det är
EXAKT samma fälla ni möter igen i M6/M7 när en molnserver ska kunna pulla
imagen utan inloggning — bra att känna igen den redan nu.

**Jag byggde på min Mac (M-serie) — fungerar imagen på klassrumsmaskinerna
eller molnservern?** Apple-kisel bygger som standard för **arm64**; de
flesta klassrumsdatorer och molnservern ni får i M7/M8 (CSC Pouta) är
**amd64**. Det stör INTE dagens lab — ni bygger och pushar, ingen pullar
tillbaka ner än. Kontrollera med `docker image inspect --format
'{{.Architecture}}' template-app-backend` om ni är nyfikna. Kom ihåg
skillnaden till M7/M8, då spelar den roll. (Bygger ni i codespacen är
frågan inte er: codespaces kör amd64, samma arkitektur som molnservern.)

**Min `.dockerignore`-PR kan inte mergas / ingen review.** Samma regler
som M2: är ni ett par kräver branch protection en godkänd review — be er
buddy titta. Solo: *Required approvals* är 0, så granska er egen PR som i
M2 (beskrivningen säger hur ni testade + minst en radkommentar) och merga.
