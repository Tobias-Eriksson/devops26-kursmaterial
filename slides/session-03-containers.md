---
marp: true
theme: default
paginate: true
title: "Session 3 — Containers"
footer: "Från commit till produktion · Session 3 · ti 15.9.2026"
---

<!-- _class: lead -->
<!-- _footer: "" -->
<!-- _paginate: false -->

# Containers

## Session 3 · Images, Dockerfiles, docker compose, registries

Mjukvaruutvecklingsprocessen — DevOps
Arcada · 8.9–20.10.2026

<!--
OBS eftermiddagspass — tisdag 13:00, samma variant som S1 (S2 var torsdag
09:15). Schemat varierar hela kursen — tisdagar, torsdagar, fredagar, en
onsdag och en måndag — så säg det rakt ut: kolla alltid kursplanens
tabell. Nästa gång är fredag 18.9 13:00.

Öppna med M2-avstämningen (nästa slide) innan något annat. Dagens session
är kursens första "bygga"-session på riktigt: appen de äger sedan M1 får
sina egna Dockerfiles och blir något de kan köra IDENTISKT på vilken
maskin som helst.
-->

---

## Läget: resultat från M2?

- Branch protection är på: en direkt push till `main` avvisas
- Minst en mergad PR per person — par: med godkänd review av buddyn · solo: med dokumenterad självgranskning i PR:en
- Commit med taggen `m2-review` är uppe på GitHub

**Fastnade du?** Fråga hjälp av läraren under labben. M1 och M2 är krav för M3.


---

## Agenda idag

1. Läget efter M2
2. **Föreläsning:** containers, images, Dockerfile, YAML, compose, registries
3. **M2**: Genomgång av M2
4. Paus
5. **Lab M3** — Dockerfiles, docker compose, push till GHCR
6. Wrap-up: verifiera M3, vanliga problem, nästa gång

<!--
Ordningen för passet, enligt kursens normala upplägg (kort föreläsning,
tyngdpunkten på labben — idag något komprimerad wrap-up till förmån för
en bred lab):
- M2-avstämning + tankenöten.
- Föreläsning: varför containers (kontra VM), images
  och lager, en annoterad genomgång av APPENS RIKTIGA Dockerfiles, YAML på
  5 minuter (precis innan compose-filen visas), docker compose, registries
  och GHCR.
- Paus (ca 10 min).
- Lab M3 enligt labs/m3-container.md. Gå runt. Största
  stoppen brukar vara GHCR-inloggning (fel scope på token) och portkrock
  (8080 redan upptagen av något annat).
- Wrap-up: verifiera-checklistan, vanliga problem,
  teaser för S4.

Dockerfilerna finns REDAN färdiga i template-repot (de ärvde dem i M1) —
dagens lab är INTE "skriv en Dockerfile från noll", utan en annoterad
genomgång + en liten egen utökningsuppgift. Säg det tydligt så ingen
förväntar sig att skriva allt själv, och ingen känner sig lurad av att
filerna redan finns.
-->

---

<!-- _class: lead -->

# Del 1: Varför containers?

<!--
Föreläsningsdelen, fram till pausen. Håll den praktisk — allt
händer på riktigt i dagens lab.
-->

---

## "Det fungerar på min dator"

Grundproblemet: en app beror på **mer än sin egen kod** — Python-version,
installerade paket, miljövariabler, operativsystem. Allt det är osynligt
och olika på varje dator.

- En **container** paketerar appen **och hela dess körmiljö** i en enda fil
- Samma image körs **identiskt** på din bärbara, klasskompisens, byggservern
  och (från S7) en molnserver
- Det är precis samma problem Git löste för koden (M1–M2) — nu löser vi det
  för **allt som koden behöver för att köra**

<!--
Koppla explicit till tankenötens svar på tavlan: varenda orsak de
räknade upp (fel Python-version, saknat paket, fel OS) är en variant av
"min dators osynliga tillstånd". En container gör det tillståndet
explicit och delbart — precis som ett repo gjorde koden delbar i M1.

Det här är dagens ramberättelse: från och med idag är "det fungerar på min
dator" inte längre ett giltigt svar i den här kursen. Om det fungerar i en
container fungerar det överallt containern körs.
-->

---

## Containers eller virtuella maskiner

![width:1050px](assets/containers-vs-vms.svg)

<!--
Bilden är dagens andra stora poäng. Gå igenom vänster till höger:

VM: varje app släpar med ett HELT gästoperativsystem ovanpå en hypervisor
— GB:s stort, startar på minuter, tungt att flytta runt.

Container: delar värdens kernel via container-motorn (Docker) — bara
applikationen och dess beroenden är paketerade, inget eget OS. MB:s stort,
startar på sekunder.

Viktig nyans om någon frågar "är det då osäkrare?": containers isolerar
med kernel-namespaces/cgroups (processnivå), VM:ar isolerar med
hårdvaruemulering (starkare gräns). För DENNA kurs och de flesta verkliga
webbappar räcker container-isolering gott — VM:ar används fortfarande där
man behöver helt olika kärnor (t.ex. Windows-gäst på Linux-värd) eller
extra stark isolering mellan hyresgäster i molnet. Nämn kort, fördjupa
inte — kursens fokus är containers.

Praktisk konsekvens de känner igen om en stund: `docker compose up` på
appen de redan äger startar på sekunder — jämför mentalt med hur länge en
VM skulle tagit.
-->

---

## Images och lager

En **image** är en stack av **read-only lager**, staplade ovanpå varandra —
inte en enda stor fil.

- Varje instruktion i en `Dockerfile` skapar (nästan alltid) **ett nytt
  lager**
- Lager är **cachade**: ändrar du inget i ett steg, återanvänder Docker
  det cachade lagret i stället för att göra om jobbet
- Ändrar du en rad i Dockerfilen byggs **det lagret och allt EFTER det**
  om — allt före förblir cachat

```bash
docker build -t template-app-backend ./backend
docker history template-app-backend   # se lagren med egna ögon
```

<!--
Samma avmystifiering som "branch är bara en pekare" i S2 — en image är
bara en lista av lager, och `docker history` gör det synligt (uppmana att
köra den i labben, bra felsökningsvana).

Nästa slide gör cache-regeln konkret med ett exempel som spelar roll på
riktigt: ORDNINGEN på raderna i Dockerfilen avgör hur mycket som byggs om
varje gång — det är skillnaden mellan en `docker build` på en sekund och
en på två minuter.
-->

---

## Cache-fällan: ordning spelar roll

![width:1050px](assets/docker-layers.svg)

<!--
Den viktigaste praktiska poängen i föreläsningen — ta tid.

Vänster (bra ordning, en förenklad version av appens backend/Dockerfile):
`requirements.txt` kopieras in och `pip install` körs INNAN appkoden
kopieras in. Ändrar du `app/main.py` påverkas bara de sista två lagren —
det dyra `pip install`-steget förblir cachat.

Höger (fallgropen, en vanlig nybörjarordning): `COPY . .` kopierar ALLT —
kod och requirements.txt — i SAMMA steg, före `pip install`. Nu ogiltig-
förklarar VARJE kodändring det steget, och `pip install` körs om från
scratch varje gång, även om inte ett enda paket ändrats. På en stor app med
tunga beroenden är skillnaden minuter per build.

Regel att ta med sig: lägg det som ändras SÄLLAN (beroenden) tidigt i
Dockerfilen, det som ändras OFTA (er egen kod) sist. Appens riktiga
Dockerfile (nästa slide) är redan byggd på det sättet — peka ut det när ni
läser den tillsammans om en stund.
-->

---

<style scoped>
section { font-size: 26px; }
</style>

## Dockerfile-anatomi: backend

Appens **riktiga** `backend/Dockerfile` — redan i ert repo sedan M1:

```dockerfile
FROM python:3.12-slim

# OS-patchning (hör till Spår C)
RUN apt-get update && apt-get -y upgrade && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
COPY pyproject.toml .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app
COPY tests ./tests

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

<!--
Läs raderna en i taget, koppla varje till föregående slides:

FROM — startpunkten, en basimage (Python 3.12 på en minimal Debian-bas,
"slim" = utan onödiga verktyg, mindre yta att hålla säker).

RUN apt-get ... upgrade — OS-patchning av basimagen, tidigt så att lagret
cachas separat före app-lagren. Säg bara "säkerhetsuppdateringar, Spår C
förklarar varför" och gå vidare — den långa kommentaren i filen är för
Spår C.

WORKDIR — sätter arbetskatalogen inuti containern; allt som följer
(COPY:er, RUN, CMD:s relativa sökvägar) utgår härifrån.

COPY requirements.txt . + COPY pyproject.toml . + RUN pip install — precis
ordningen från föregående slide: beroenden (och ruffs config, som knappt
ändras) separat, INNAN koden, för cachens skull. `pyproject.toml` behövs
inte av `pip install` — den läses av `ruff` när ni kör linting — men den
ändras lika sällan som `requirements.txt`, så den hör hemma i samma
tidiga lager. `--no-cache-dir` är en pip-flagga (inte Dockers cache) som
undviker att pip:s egen nedladdningscache blir kvar i imagen — mindre
slutstorlek.

COPY app ./app + COPY tests ./tests — koden sist, av samma cache-skäl.

EXPOSE 8000 — REN DOKUMENTATION. Den publicerar INTE porten själv (vanligt
missförstånd) — det gör `-p`/`ports:` när containern körs, vilket vi ser i
compose-filen om en stund.

CMD — kommandot som körs när containern STARTAR (inte när den byggs).
Array-formen (`["uvicorn", ...]`) körs direkt utan ett shell emellan —
nämn kort, fördjupa inte.

Fråga att ställa: "vad tror ni händer om ni byter plats på COPY app ./app
och RUN pip install?" — låt dem svara utifrån föregående slide innan ni
går vidare.
-->

---

## Dockerfile-anatomi: frontend

Appens **riktiga** `frontend/Dockerfile` — kortare, statisk nginx-server:

```dockerfile
FROM nginxinc/nginx-unprivileged:alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY index.html style.css app.js /usr/share/nginx/html/

EXPOSE 8080
```

- **frontend-imagen** (`FROM nginxinc/nginx-unprivileged:alpine`) — ~90 MB
  mot **backend-imagen** ~340 MB
- Ingen `RUN`-rad: nginx **är** hela körmiljön
- Inget eget `CMD`: nginx-imagens **standard-CMD** startar servern
- **Icke-root** basimage — lyssnar på 8080, inte 80

<!--
Alpine-varianten av nginx-imagen är en minimal distribution byggd för
just små images —
därav skillnaden mot backendens ~340 MB. Ingen egen RUN-rad behövs
eftersom nginx redan är hela körmiljön, vi lägger bara in konfiguration
och statiska filer.

Kortare Dockerfile eftersom frontend inte har egna beroenden att
installera — nginx-imagen GÖR redan jobbet, vi konfigurerar bara. Bra
kontrast mot backendens Dockerfile: visar att en Dockerfiles längd speglar
hur mycket den faktiskt behöver göra, inte ett mall-krav.

Alpine-notan: musl i stället för glibc, minimal paketuppsättning — därav
den lilla storleken. Nämn kort att alpine ibland ger
kompatibilitetsöverraskningar för Python-paket med C-beroenden (därför backendens bas är
`slim`, inte `alpine`) — fördjupa inte, det är en avvägning kursen redan
gjort åt dem.

`nginx.conf` (kort titt, finns i labben): en `location /api/` som proxar
till `backend:8000` — DÄR namnet `backend` kommer från compose-filens
service-namn, se nästa del.

Icke-root-notan (30 sekunder, ta den — den kommer tillbaka i Spår B):
vanliga `nginx`-imagen kör som root och lyssnar på 80. Lokalt i Docker
märks det inte, men de flesta seriösa driftplattformar vägrar köra
containers som root — vår egen frontend kraschade med "Permission denied"
på Rahti innan vi bytte bas. Portar under 1024 kräver root, därför
lyssnar den icke-privilegierade imagen på 8080 i stället. Poängen för
dem: valet av basimage är ett driftbeslut, inte bara en storleksfråga.
-->

---

<!-- _class: lead -->

# YAML på 5 minuter

## Allt ni behöver för resten av kursen

<!--
Läraranteckning (bindande för den här sessionen): det här är studenternas
FÖRSTA möte med YAML i kursen — nästa slide är docker-compose.yml, som är
skriven i YAML. Kör det här blocket (3 slides, ~5 min) INNAN
compose-filen visas, inte efter.

Rama in stort: det här är inte en Docker-detalj, det är en investering som
betalar sig FYRA gånger till i kursen — docker-compose.yml (strax),
Actions-workflows (M5–M6), cloud-init (M8), och k8s-manifest (spår B).
Samma syntax, om och om igen. Lär er den EN gång, ordentligt, nu.

Var samtidigt tydlig om vad "lära er" betyder: ingen memorerar ett
YAML-schema utantill — varken compose-nycklar, Actions-nycklar eller
k8s-fält. Målet är att kunna LÄSA en YAML-fil, förstå vad den gör, och
veta var referensen finns (docs, --help, exempel i det här repot). Det
kommer alltid finnas mer konfiguration än man minns — det är normalt,
inte ett tecken på att man inte fattat.
-->

---

## YAML: indrag och kartor

- **Indrag ÄR syntax** — YAML har inga klamrar `{}` eller `end`, bara
  **mellanslag**. Fel indrag = fel struktur, eller ett fel som stoppar hela
  filen
- **Använd ALDRIG tabbar** — YAML-specen tillåter dem inte för indrag; de
  flesta editorer (VS Code inkluderat) kan ställas in att infoga
  mellanslag när du trycker Tab
- **`nyckel: värde`** är grundbygget — en karta (map) av namngivna fält

```yaml
tjänst:
  namn: backend
  port: 8000
```

<!--
Timing: ~1.5 min. Det viktigaste att landa: YAML har INGA visuella
klamrar som Python eller JSON — strukturen ÄR indraget, osynligt om man
inte tittar noga. Det är källan till nästan alla YAML-buggar hela kursen
igenom: en rad som ser rätt ut men har fel antal mellanslag.

Tabb-varningen är konkret och sparar frustration: många editorer infogar
tabbar som standard. VS Code (deras devcontainer/Codespace) har
YAML-extensionen (redhat.vscode-yaml) förinstallerad — den varnar rött vid
tabb-indrag. Peka ut det i labben om någon fastnar.

Exemplet ovan: `tjänst` är en karta med två nycklar, `namn` och `port`,
indragna två steg under `tjänst:`. Ett konsekvent indrag (två mellanslag
här) håller ihop kartan.
-->

---

## YAML: listor och citattecken

**Listor** börjar med `- ` (bindestreck + mellanslag), på samma indragsnivå:

```yaml
portar:
  - 8080
  - 8443
```

**Citera strängar** som annars kan misstolkas — klassikern är
**Norge-problemet**: `no`, `yes`, `on`, `off` (även `NO`, `Yes`, ...) tolkas
som **booleaner**, inte text, om de inte citeras:

```yaml
land: no        # blir booleanen false, INTE landskoden för Norge!
land: "no"      # rätt — nu är det texten "no"
```

<!--
Timing: ~2 min, den roligaste och mest minnesvärda regeln.

Norge-problemet är en riktig, dokumenterad YAML-fälla (YAML 1.1, som
många parsrar — inklusive äldre Compose-versioner — fortfarande följer):
landskoden för Norge är bokstavligen "NO", och en ociterad "NO" blir
boolean-falskt. Berättelsen fastnar bättre än regeln i sig — låt den göra
jobbet.

Praktisk konsekvens de MÖTER idag: i docker-compose.yml (nästa del) är
portar skrivna som `"8080:8080"`, citerade, trots att det ser ut som en
siffra. Peka tillbaka hit när den sliden visas — det är exakt samma regel:
citera när tolkning är tvetydig.

Framåtblick värd att nämna: i Actions-workflows (M5) skriver ni nyckeln
`on:` för triggers, ocitrerad, varje gång — det är just den nyckeln som
YAML 1.1 annars skulle kunna läsa som en boolean. GitHub Actions parsern
hanterar det åt er, men nu vet ni VARFÖR den varningen finns i vissa
YAML-linters.
-->

---

## YAML: flerradiga strängar

Två sätt att skriva text över flera rader:

```yaml
literal: |
  Rad ett.
  Rad två.
  (radbryten bevaras exakt)

folded: >
  Rad ett.
  Rad två.
  (blir "Rad ett. Rad två." — radbrytningar blir mellanslag)
```

`|` när formatet spelar roll (skript, kommandon). `>` när det bara är löpande
text.

<!--
Timing: ~1 min, hålls medvetet kort.
Bara igenkänning behövs nu, inte flyt.

`|` (literal/pipe) återkommer konkret i M8: cloud-init-skriptets
`runcmd`-sektion och liknande fält skrivs ofta med `|` just för att
radbrytningarna MÅSTE bevaras (det är kommandorader, inte löptext).

`>` (folded) ses mer sällan i den här kursen men är bra att känna igen —
klassisk användning är en lång beskrivningstext som man vill radbryta i
källfilen för läsbarhets skull utan att det syns i resultatet.

Avsluta blocket med löftet: "det här är sista gången vi förklarar YAML
grundligt — från och med nu FÖRUTSÄTTER kursen att ni kan det här." Gå
direkt till nästa slide, som är den första riktiga YAML-filen: compose.
-->

---

<!-- _class: lead -->

# Del 2: docker compose och registries

---

<style scoped>
section { font-size: 27px; }
</style>

## docker compose: hela appen, ett kommando

Appens riktiga `docker-compose.yml` — en YAML-karta av **tjänster**:

```yaml
services:
  backend:
    build: ./backend
    image: template-app-backend

  frontend:
    build: ./frontend
    image: template-app-frontend
    ports:
      - "8080:8080"
    depends_on:
      - backend
```

```bash
docker compose up --build   # bygg + starta båda tjänsterna
docker compose down         # stäng ner allt
```

<!--
Läs strukturen med YAML-glasögonen från nyss: `services` är en karta av
tjänstenamn → konfiguration; `backend` och `frontend` är egna kartor;
`ports` och `depends_on` är LISTOR (`- `) — samma syntax de just lärde sig,
nu på riktigt.

`build: ./backend` — bygg imagen från Dockerfilen i den mappen (samma
`docker build` som nyss, fast Compose gör det åt er för alla tjänster på en
gång).

`image: template-app-backend` — ger den lokalt byggda imagen ett fast
namn (annars döps den efter katalogen den byggs i, olika mellan
studenternas kloner); det är därför `docker tag`-kommandot på nästa
slide fungerar likadant för alla.

`ports: - "8080:8080"` — CITERAT, trots att det ser ut som siffror; värdens
port 8080 kopplas till containerns port 8080 (den EXPOSE:ade porten i
frontend-Dockerfilen). Att båda råkar vara 8080 är en bekvämlighet, inte
en regel — vänster sida är värdens port, höger sida containerns. Koppla
till Norge-regeln: citattecknen är där av
samma anledning som `"no"` behövde dem — undvik tvetydig tolkning.

`depends_on: - backend` — VIKTIG NYANS: det här styr bara STARTORDNING
(backend-containern startas före frontend), INTE att backend är REDO att
svara på anrop. Compose väntar inte på en hälsokontroll. I appens fall
räcker det (uvicorn startar snabbt) — men det är en vanlig missuppfattning
värd att nämna nu, den blir relevant igen i M9.

Tjänstenamnet `backend` är också DNS-namnet andra tjänster använder för
att nå den — det är därför `nginx.conf`:s `proxy_pass http://backend:8000/api/;`
(öppnas i labbens steg 2) fungerar: Compose skapar ett internt nätverk där
tjänstenamnen slår upp varandra automatiskt.
-->

---

## Registries: dela images

En **registry** är för images vad GitHub är för kod — ett ställe att
**pusha till** och **pulla från**, med versioner (taggar).

- CI bygger images med namnet
  `ghcr.io/<ert-användarnamn>/template-app-backend:<tagg>`
- Vi använder **GHCR** — samma GitHub-konto sedan M1, men en engångs-`docker login` med token (labben)
- Workflown `publish-images.yml` pushar redan images vid merge till
  `main`
- **Idag: för hand.** `docker login` + `docker tag` + `docker push`

```bash
docker tag template-app-backend:latest ghcr.io/<ert-användarnamn>/template-app-backend:latest
docker push ghcr.io/<ert-användarnamn>/template-app-backend:latest
```

<!--
Ramat medvetet som "ni gör för hand det workflown redan gör åt er" —
samma pedagogiska princip som M2:s branch protection (grinden fanns
innan den blev obligatorisk) och M1:s redan körande "Lint and test
backend"-check. `publish-images.yml` har körts sedan M1, precis som den
checken — ingen av dem är ny i M6. Att göra jobbet för hand EN gång
bygger förståelse för vad workflown faktiskt gör, innan ni öppnar och
äger den i M6.

`docker tag` skapar inget nytt — bara ett extra NAMN på samma image-ID,
pekande på registryn ni tänker pusha till. `docker push` skickar lagren
(kom ihåg: bara de som INTE redan finns i registryn behöver överföras —
cache-principen från tidigare gäller även här).

Inloggningsdetaljer (PAT med `write:packages`-scope) och synlighets-
fällan (nypublicerat paket är PRIVAT som standard) hör till labben — peka
dit, kommer i detalj i wrap-upens vanliga problem också.

`<ert-användarnamn>` måste skrivas med SMÅ BOKSTÄVER — Docker tillåter inte versaler
i repository-sökvägen, även om GitHub-användarnamnet har det. Nämn det
högt om någon har versaler i sitt användarnamn; labbens Steg 5 och
Vanliga problem har den fällan i detalj.

Görs en gång per image — kodblocket visar backend, samma två kommandon
upprepas för frontend-imagen.
-->

---

<!-- _class: lead -->

# Lab: M3

## Dockerfiles · docker compose · push till GHCR

<!--
Här börjar labben. Instruktionen finns i labs/m3-container.md. Säg tydligt: Dockerfiles
och docker-compose.yml finns REDAN i repot (ärvda från template i M1) —
dagens lab är en GUIDAD GENOMGÅNG (läs, förutsäg, kör, jämför) plus en
liten egen utökning, inte "skriv allt från noll". Ingen ska känna sig
lurad av att filerna redan finns — det är tvärtom poängen: läs kod som
redan fungerar, bygg förståelse, lägg sedan till något EGET.

Codespaces-fallback: nämn direkt att den som får Docker-problem lokalt
(disk fullt, Docker Desktop startar inte, WSL2-krångel) byter till en
Codespace — devcontainern har docker-in-docker inbyggt, allt fungerar
identiskt. Det är dagens session där det först är relevant på riktigt
("bara webbläsare" har hittills varit en bonus, idag kan den vara en
räddning) — labben har en egen ruta för det.
-->

---

## M3 — dagens milstolpe

**Mål:** ni förstår appens Dockerfiles, hela appen kör med `docker
compose`, och båda images finns i GHCR.

1. **Läs och förutsäg:** gå igenom de befintliga Dockerfilerna rad för rad —
   förutsäg vad som händer INNAN ni kör, jämför med verkligheten
2. **Bygg och kör:** `docker compose up --build`, verifiera i webbläsaren
3. **Egen utökning:** lägg till `.dockerignore` för båda tjänsterna
4. **Registry:** logga in, tagga, pusha båda images till GHCR (manuellt)
5. Tagga: `git tag m3-container` — och pusha taggen

**Bevis:** appen kör via compose + båda images i GHCR + taggen uppe + osynligt arbete dokumenterat i `inlamning/m3-container.md`.

<!--
Bevis-kravet i korthet: compose-körningen och GHCR-paketen syns inte i
repot ("det osynliga arbetet"), så några rader om dem i inlamningen
räcker som dokumentation.

Kör labben i instruktionens ordning. Steg 1 (läs-och-förutsäg) är
medvetet placerat FÖRE byggandet — låt dem gissa ("varför kommer COPY
requirements.txt före COPY app?") innan de kör och ser svaret.
Det är samma pedagogik som föreläsningens cache-fråga.

Att gå runt och titta efter:
- GHCR-inloggning: vanligaste stoppet är fel scope på personal access
  token (måste ha `write:packages`) — detaljerat i labbens problemsektion.
- Portkrock: 8080 redan upptagen av något annat lokalt program — labben
  har lösningen (ändra host-porten i compose, INTE container-porten).
- .dockerignore-uppgiften: påminn om M2:s löfte — även denna lilla
  ändring går via branch + PR + review, inte direkt commit till main.
- Glömt `docker push` av BÅDA images (bara en av två) — vanligt när man
  har bråttom, kolla GHCR:s Packages-sida har två poster.
-->

---

## Wrap-up: är M3 klar?

- [ ] `docker compose up --build` startar **båda** tjänsterna utan fel
- [ ] Appen svarar i webbläsaren på **localhost:8080**
- [ ] `.dockerignore` för båda tjänsterna finns, mergade via **granskad PR**
- [ ] Er egen `:latest`-push syns med **färsk tidsstämpel** för **båda** paketen
      (`template-app-backend`, `template-app-frontend`) under **Packages**
- [ ] Taggen `m3-container` syns under **Tags**, och det **osynliga arbetet** är dokumenterat i `inlamning/m3-container.md`

Fastnade du? Ta det **olöst** till session 4 — M1–M9 bör vara klara till session 10.

<!--
Gemensam avstämning — kör checklistan tillsammans. Vanliga problem att nämna högt (fulla detaljer
i labbens problemsektion — peka dit i stället för att lösa allt live):

- **Portkrock (`bind: address already in use`)** — något annat lokalt
  program lyssnar redan på 8080. Ändra HOST-porten i compose
  (`"8081:8080"`), inte container-porten.
- **Cache-överraskningar** — "min build tar lika lång tid varje gång" är
  nästan alltid fel ordning i Dockerfilen (dagens föreläsningspoäng på
  riktigt). Kolla ordningen mot backend-Dockerfilen.
- **arm64/amd64** — bygger du på en Mac med Apple-kisel (M-serien) bygger
  Docker som standard för **arm64**. De flesta klassrumsmaskiner (Intel)
  och molnservern ni får i M7/M8 (CSC Pouta) är **amd64**. Det stör INTE
  dagens lab (ni kör och pushar lokalt, ingen pullar tillbaka ner än) —
  men kom ihåg det till M7/M8, då spelar det roll. `docker image inspect
  --format '{{.Architecture}}' template-app-backend` visar vilken
  arkitektur en image byggdes för.
- **GHCR-inloggning/synlighet** — token utan `write:packages`-scope ger
  `denied` vid push; ett nypublicerat paket är dessutom PRIVAT som
  standard (samma fälla som M6/M7 möter igen, i detalj i labben).

Kulturpåminnelsen: `.dockerignore`-ändringen gick via PR precis som allt
annat sedan M2 — normalisera att ÄVEN en enrads-fil följer arbetssättet.

Tips till paren: dela upp .dockerignore-arbetet (en tar backend, en
frontend) och korsgranska PR:arna — labben rekommenderar det i steg 4,
ett muntligt nudge hjälper de som missat det. Solo: självgranskningen
från M2 (beskrivning + egen radkommentar), inte någon buddy.
-->

---

## Nästa gång

**Session 4 · fr 18.9 kl 13:00 — Automatisk testning**

Testpyramiden, `pytest`, `ruff`, coverage.

**Ta med:** ett M3-klart repo — containers ni förstår, images i GHCR.

> Idag paketerade ni appen. Nästa gång bevisar ni att den faktiskt gör rätt
> sak — automatiskt, varje gång.

<!--
OBS eftermiddagspass (13:00, som S3) men ny veckodag — fredag i stället för
tisdag — påminn kort.

Teasern: S4 är kursens första riktiga testnings-session. Testerna följer
redan med in i backend-imagen (peka tillbaka på `COPY tests ./tests`-raden
om någon undrar varför) — S4 fördjupar det och lägger till en medveten bugg de ska HITTA
via ett test, inte gissa fram.

Sista ordet: den som inte fick `m3-container`-taggen uppe — gör klart före
fredag, be om hjälp via itslearning eller mejla läraren. M4 bygger vidare på samma repo, inget
nytt att sätta upp.
-->
