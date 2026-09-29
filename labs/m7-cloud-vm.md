# M7 – VM i cPouta för hand, security group, floating IP, SSH, Docker, appen live

Den här labben hör till **session 7** (Molnet manuellt). Ni jobbar i par
(eller solo), ca 90 minuter.

**Milstolpens bevis:** en VM kör i CSC cPouta med en associerad floating
IP, security groupen tillåter port 22 (SSH) och 8080 (appen), Docker
kör båda images pullade från GHCR, appen svarar på floating-IP:n
**utifrån** — från er codespace eller er egen dator, inte bara från
VM:en själv — och nip.io-URL:en visar appen i en webbläsare. 5 skärmdumpar finns i
`inlamning/m7-cloud.md`, mergat via en egen PR, och taggen `m7-cloud` är
uppe.

**Viktigt att veta innan ni börjar:** allt idag görs **manuellt** i
webbkonsolen. Det är medvetet klumpigt — i session 8 (M8) låter ni
Terraform bygga upp exakt samma sak automatiskt, och river sedan den
handbyggda VM:en. Skriv gärna ner
varenda inställning ni väljer idag (image, flavor, security group-regler,
key pair), ni kommer jämföra listan mot en Terraform-fil om två dagar.

**Om cPouta-konsolen (Horizon):** steg som görs i webbkonsolen är
märkta **[KONSOL]** i rubriken. Steg 2–5 (2026-08-12) är körda av
läraren i den riktiga konsolen och
skärmdumparna i de stegen kommer från den körningen — menytexter,
fältnamn och knappar stämmer alltså med det ni ser. Webbkonsolen kan
ändå ha ändrats sedan dess: ser er vy annorlunda ut, följ bilden i första hand
och säg till.

## Steg 0 – Förkrav

- [ ] Uppdatera kursmaterialet: `cd course-material && git pull`.
- [ ] Öppna er codespace (eller er lokala klon) — labben körs i en
      terminal där, och SSH-sessionen mot VM:en startas därifrån.
- [ ] M1–M3, M5 och M6 är klara: taggarna `m1-repo`, `m2-review`,
      `m3-container`, `m5-ci` och `m6-cd-images` är uppe. Båda
      GHCR-paketen (`template-app-backend`, `template-app-frontend`) är
      **Public** (`labs/m6-cd-images.md`, steg 4). Utan det svarar VM:ens
      `docker compose pull` senare med `denied`.
- [ ] M4 krävs inte för dagens steg — men gör klart det innan session 10.
- [ ] Rulesetet från M2 + M5 är på: **Enforcement status: Active**,
      **Require a pull request before merging**, *Required approvals*
      **1 om ni är ett par, 0 om du jobbar solo**, och **Require status
      checks to pass** med `Lint and test backend`. Dagens inlämnings-PR
      går igenom exakt samma spärr.
- [ ] Checken `Lint and test backend` var **grön** på er senaste mergade
      pull request. Var den **röd**? Då är `main` redan röd och ingen PR
      i dag kan mergas — se Vanliga problem i `labs/m5-ci.md` först.
- [ ] Ert **eget CSC-projekt** finns i MyCSC, cPouta är aktiverat på det
      och **båda i paret** är medlemmar via Haka (ordnades efter session
      1 — se `labs/m1-forsta-commit.md`, avsnittet "Efter labben").
      Kommer någon av er inte in på <https://pouta.csc.fi> med sin
      Haka-inloggning: kolla i MyCSC att medlemsansökan är godkänd och
      att var och en godkänt Pouta-villkoren för egen del. Löser det inte
      saken är det ett administrativt ärende — fråga läraren innan ni går
      vidare, gissa er inte fram.
- [ ] En terminal med `ssh-keygen`, `ssh` och `curl`. **Kör ni i
      codespacen är allt redan på plats** (devcontainern installerar
      `openssh-client`) — hela labben går att göra därifrån. Lokalt:
      macOS/Linux/WSL har detta redan, klassrumsmaskinerna med Git Bash
      också.
- [ ] **Par:** var och en gör steg 1 för egen del. Den vars nyckel VM:en
      startas med klickar konsolstegen 2–5 (och laddar upp bådas publika
      nycklar i steg 2) medan den andra skriver ner
      varje inställning (listan behövs i M8), och loggar in först i
      steg 6. Efter steg 7 tar den andra över på VM:en (steg 8–10) och
      öppnar inlämnings-PR:en i steg 11 — den första granskar och
      godkänner den.
- [ ] **Solo:** hoppa över steg 7 (ingen partner-nyckel att lägga in).
      *Required approvals* står kvar på **0** — du granskar din
      PR som i M2 och M5 (beskrivning + minst en egen radkommentar).

## Steg 1 – Skapa ett SSH-nyckelpar var (i er egen miljö, verifierbart)

**Var och en av er** skapar ett eget dedikerat nyckelpar i sin egen
arbetsmiljö — codespacen eller den egna datorn — ni delar alltså inte
ett nyckelpar mellan er. Kör var för sig (solo: ett nyckelpar räcker):

```bash
ssh-keygen -t ed25519 -C "din@epost" -f ~/.ssh/cpouta_ed25519
```

**Sätt en lösenfras** när kommandot frågar (inte tom). Lösenfrasen är det
enda som skyddar nyckeln om filen hamnar fel — och den blir extra viktig
om ni förvarar nyckeln på USB eller i molnlagring (se noten nedan).
Slipp skriva den vid varje inloggning genom att ladda nyckeln i
ssh-agenten en gång per session:

```bash
ssh-add ~/.ssh/cpouta_ed25519
```

**I codespacen:** ingen ssh-agent körs som standard, så `ssh-add` svarar
`Could not open a connection to your authentication agent`. Starta en
agent först — men den lever bara i den terminalflik ni startade den i,
så det får göras om i varje ny terminal:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/cpouta_ed25519
```

Vill ni slippa det helt: hoppa över agenten. `ssh -i` (som stegen nedan
använder) fungerar alltid, ni skriver bara lösenfrasen vid varje
anslutning.

Kontrollera sedan, var och en, att båda filerna finns och att den privata
nyckeln har rätt rättigheter:

```bash
ls -l ~/.ssh/cpouta_ed25519 ~/.ssh/cpouta_ed25519.pub
chmod 600 ~/.ssh/cpouta_ed25519
```

**Regeln:** `cpouta_ed25519` (utan `.pub`) är **privat** — den laddas ALDRIG
upp till CSC, ett repo, en chatt eller ett AI-verktyg, och delas
**aldrig** med någon annan — inte ens med er par-kompis. Att ni båda ska in på samma VM löses inte genom att dela
en privat nyckel, utan genom att båda **publika** nycklarna installeras
på VM:en (steg 7).

**Ingen egen dator?** Nyckeln måste överleva att klassrumsprofilen rensas
eller att en codespace raderas. Förvara den privata nyckeln — som alltid
med lösenfras — i er Arcada-OneDrive, eller på ett USB-minne.

## Steg 2 – [KONSOL] Ladda upp era publika nycklar till cPouta

Ladda gärna upp **bådas** publika nycklar — då syns det i listan vems
nyckel som är vems. Vid VM:ens start kan OpenStack ändå bara injicera
**en** av dem; den andra i paret kommer in via steg 7.

1. Logga in på <https://pouta.csc.fi> med Haka, välj ert projekt i
   projektväljaren uppe till vänster.
2. **Compute → Key Pairs**. På sidan finns två knappar: **Create Key
   Pair** (låter Horizon skapa nyckeln åt er — inte det ni ska göra) och
   **Import Public Key** (laddar upp en nyckel ni redan har). Klicka
   **Import Public Key**.
3. Namnge nyckeln med ert eget namn (t.ex. `<förnamn>-key`), klistra in
   innehållet i `~/.ssh/cpouta_ed25519.pub` (öppna filen med
   `cat ~/.ssh/cpouta_ed25519.pub` och kopiera hela raden).
4. Bekräfta att nyckeln syns i listan, med **Type** `ssh` och ett
   **Fingerprint**. Fingeravtrycket är nyckelns kvitto — det är så ni
   ser vems nyckel som är vems om namnen är otydliga.
5. Bestäm vems nyckel VM:en ska startas med — den väljer ni i steg 4.

![Key Pairs-vyn i Horizon: knappen Import Public Key uppe till höger och
den uppladdade nyckeln i listan med kolumnerna Name, Type och
Fingerprint](assets/m7-keypairs.png)

> 📸 **Kom ihåg skärmdump till inlämningen:** Key Pairs-listan med er nyckel (båda, i par), **Type** `ssh` och **Fingerprint** synliga.

**Kom ihåg:** keypair-valet i Horizon injiceras bara vid VM:ens **start**
och går inte att ändra i efterhand — men det gäller bara den metadatan,
inte åtkomsten till VM:en. **Nycklar läggs till fritt efteråt** genom att
skriva in fler publika nycklar i `~/.ssh/authorized_keys` på VM:en (steg
7). Ny VM behövs bara om ni tappar **alla** nycklar som finns installerade.

## Steg 3 – [KONSOL] Skapa en security group

Standard-security groupen blockerar allt inkommande. Skapa en egen (eller
utöka default) med två regler:

1. **Network → Security Groups → Create Security Group**, namnge den
   (t.ex. `m7-app`).
2. Öppna gruppen → **Add Rule**. Dialogen ser olika ut beroende på vad
   ni väljer i **Rule**-listan högst upp — det är den vanligaste
   förvirringen här.
3. **Regel 1 (SSH):** välj **Rule = SSH**. Det är en färdig mall: den
   fyller i port 22 och riktningen Ingress åt er, så dialogen visar
   varken något portfält eller något **Direction**-fält. Det enda ni
   sätter är **Remote = CIDR** och **CIDR** `0.0.0.0/0` (eller er egen
   IP med `/32` om ni vill begränsa — men SSH:ar ni från en codespace
   passar `/32` inte: GitHubs käll-IP byts mellan sessioner, och regeln
   låser er ute). Klicka **Add**.

   ![Add Rule-dialogen med Rule satt till SSH: bara fälten Description,
   Remote (CIDR) och CIDR 0.0.0.0/0 syns — inget Direction-fält, för
   SSH-mallen är alltid Ingress](assets/m7-sg-rule-ssh.png)

4. **Regel 2 (appen):** **Add Rule** igen, välj **Rule = Custom TCP
   Rule**. Nu dyker fler fält upp: sätt **Direction = Ingress**,
   **Open Port = Port**, **Port** `8080`, **Remote = CIDR**, **CIDR**
   `0.0.0.0/0`. Klicka **Add**. 8080 är porten frontend-containern
   publiceras på (8080:8080, samma mapping som i M3).

   ![Add Rule-dialogen med Rule satt till Custom TCP Rule: Direction
   Ingress, Open Port Port, Port 8080, Remote CIDR och CIDR
   0.0.0.0/0](assets/m7-sg-rule-8080.png)

5. Dubbelkolla riktningen på regel 2: **Ingress** (in mot VM:en), inte
   **Egress** (ut från VM:en, redan öppet som standard och inte det ni
   behöver ändra). Båda reglerna ska nu synas i gruppens regellista.

> 📸 **Kom ihåg skärmdump till inlämningen:** gruppens regellista med båda reglerna — port 22 och 8080, **Ingress**.

## Steg 4 – [KONSOL] Launch Instance

**Compute → Instances → Launch Instance** öppnar en guide med flikarna
i vänsterkanten: **Details, Source, Flavour, Networks, Network Ports,
Security Groups, Key Pair, Configuration, Server Groups, Metadata**.
Flikar märkta med asterisk är obligatoriska. På flikarna Source,
Flavour, Networks, Security Groups och Key Pair väljer ni genom att
klicka **pil upp (↑)** på raden ni vill ha i listan **Available** —
raden flyttas då upp till **Allocated**. Det räcker alltså inte att
klicka på raden.

1. **Details:** instansnamn, t.ex. `m7-<förnamn>`.
2. **Source:** Flytta upp **Ubuntu-24.04** till
   Allocated med pilen. Standardanvändaren för Ubuntu-images på cPouta
   är `ubuntu`.

   ![Launch Instance, fliken Source: Select Boot Source satt till Image,
   Create New Volume satt till No, och Ubuntu-24.04 upplyft till listan
   Allocated](assets/m7-launch-source.png)

3. **Flavour:** välj **`standard.small`** (2 VCPU, 1.95 GB RAM, 80 GB
   disk) — `standard.tiny` (1 VCPU, 1000 MB RAM) är för litet för en
   två-containers app. Rader med en orange varningstriangel är större än
   projektets kvot och går inte att starta; fråga läraren om ni är
   osäkra.

   ![Launch Instance, fliken Flavour: listan Available med
   standard.tiny, standard.small, standard.medium och större flavours,
   där de största har orange varningstrianglar för
   kvoten](assets/m7-launch-flavour.png)

4. **Networks:** ert projekts eget nätverk ska redan ligga i Allocated.
5. **Security Groups:** flytta upp `m7-app` (steg 3) — ta bort default
   från Allocated om den bara blockerar.
6. **Key Pair:** flytta upp den nyckel ni bestämde i steg 2 (bara en går
   att välja — den andra läggs till i steg 7).
7. Klicka **Launch Instance** nere till höger. Instansen dyker upp i
   **Compute → Instances** och ska efter en stund ha **Status = Active**
   och **Power State = Running** (bilden i steg 5).

Rör inget annat i guiden: flikarna **Network Ports**, **Configuration**,
**Server Groups** och **Metadata** lämnas som de är.

## Steg 5 – [KONSOL] Associera en floating IP

En ny instans får bara en **intern** adress (`192.168.X.Y` i
IP Address-kolumnen) och är nåbar bara inifrån CSC:s nätverk tills en
floating IP kopplas till den.

1. **Compute → Instances** → hitta er instans. Längst till höger på
   raden står knappen **Create Snapshot** med en liten pil bredvid —
   klicka **pilen** och välj **Associate Floating IP** i menyn som
   fälls ut.

   ![Instances-listan med instansen m7-topi som Active och Running,
   flavor standard.small, och den utfällda Actions-menyn där Associate
   Floating IP är översta valet](assets/m7-instances-actions.png)

2. Dialogen **Manage Floating IP Associations** öppnas. Står det **No
   floating IP addresses allocated** i fältet **IP Address** har ert
   projekt ingen ledig IP — klicka **+**-knappen bredvid fältet för att
   allokera en från poolen.

   ![Dialogen Manage Floating IP Associations där IP Address visar No
   floating IP addresses allocated och plusknappen bredvid fältet
   allokerar en ny adress](assets/m7-floating-ip-none.png)

3. Dialogen **Allocate Floating IP** öppnas. **Pool** är förvalt till
   **PUBLIC** — lämna det, Description är valfritt, och klicka
   **Allocate IP**. Kvotstapeln **Project Quotas** visar hur många
   floating IP:er projektet redan använder av sin kvot.

   ![Dialogen Allocate Floating IP med Pool satt till PUBLIC, ett
   tomt Description-fält, kvotstapeln Floating IP 1 of 2 Used och
   knappen Allocate IP](assets/m7-floating-ip-allocate.png)

4. Välj den allokerade adressen i **IP Address**, kontrollera att
   **Port to be associated** är er instans (namnet plus dess interna
   adress), och klicka **Associate**.

   ![Samma dialog efter allokeringen: IP Address visar den publika
   adressen 195.148.30.206 och Port to be associated visar instansen med
   sin interna adress](assets/m7-floating-ip-selected.png)

5. Instansradens **IP Address**-kolumn ska nu visa både den interna
   (`192.168.X.Y`) och den publika floating-adressen.

> 📸 **Kom ihåg skärmdump till inlämningen:** instansraden i **Compute → Instances** med **Active**, **Running** och båda adresserna (intern + floating IP).

Notera den publika IP:n, t.ex. `195.148.X.Y` — den behövs i alla
följande steg. Blanda inte ihop den med den interna `192.168.X.Y`, som
inte går att nå utifrån — varken från er codespace eller er egen dator.

## Steg 6 – SSH in

Den av er vars nyckel VM:en startades med (steg 4) loggar in först, från
sin egen arbetsmiljö. **SSH fungerar i codespacen** — klienten finns
installerad och utgående port 22 är öppen, så ni behöver inte byta till
en lokal terminal för det här:

```bash
ssh -i ~/.ssh/cpouta_ed25519 ubuntu@<er-floating-ip>
```

Första gången frågar SSH om ni litar på värdens fingeravtryck — svara
`yes`. Kommer ni inte in: se **Vanliga problem** längst ner (SG-riktning
och nyckelrättigheter är de vanligaste orsakerna).

## Steg 7 – Lägg in partnerns publika nyckel på VM:en

Just nu kommer bara en av er in. Fixa det direkt, innan ni gör något
annat på VM:en. **Solo:** hoppa över steget — men läs "Varför det här är
viktigt" nedan.

Den som **inte** kom in kör i sin egen arbetsmiljö och kopierar hela
raden:

```bash
cat ~/.ssh/cpouta_ed25519.pub
```

Den som är inloggad på VM:en klistrar in raden och lägger till den i
`authorized_keys` (`>>` lägger TILL — använd inte `>`, det skriver över
nyckeln ni redan kom in med):

```bash
echo '<partnerns publika nyckel>' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Verifiera från den andras arbetsmiljö — nu ska **båda** komma in med sin
egen nyckel:

```bash
ssh -i ~/.ssh/cpouta_ed25519 ubuntu@<er-floating-ip>
```

**Varför det här är viktigt:** `~/.ssh/authorized_keys` är listan över
vilka publika nycklar som får logga in som användaren `ubuntu`. Tappar
**en** av er sin nyckel (rensad klassrumsprofil, raderad codespace,
borttappat USB) loggar den andra in och lägger till en ny publik nyckel
på exakt samma sätt — ingen ny VM behövs. Först när **bådas** nycklar är
borta står ni utan väg in och måste bygga om.

**Förhandsvisning av M9:** det ni just gjorde är precis samma mekanik som
deploy-användaren i M9 — där lägger ni in en dedikerad deploy-nyckels
publika del i `authorized_keys` så att GitHub Actions kan SSH:a in. Åtkomst
ges alltid genom att lägga till **publika** nycklar, aldrig genom att dela
en privat.

## Steg 8 – Installera Docker

På VM:en (via SSH-sessionen från steg 6):

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

Kontrollera:

```bash
sudo docker --version
sudo docker compose version
```

Båda ska skriva ut en version. Glöm inte `sudo`: användaren `ubuntu` är
inte med i `docker`-gruppen, så utan `sudo` svarar Docker `permission
denied … docker.sock`. Alla Docker-kommandon i labben körs med `sudo`.

(`get.docker.com`-skriptet installerar docker-ce, cli, containerd och
compose-plugin i ett svep — samma installationskommando används av
kursens cloud-init-fil för M8, ni kör bara stegen för hand idag.)

## Steg 9 – Skriv docker-compose.yml och starta stacken

Fortfarande på VM:en:

```bash
sudo mkdir -p /opt/app
sudo tee /opt/app/docker-compose.yml > /dev/null <<'EOF'
services:
  backend:
    image: ghcr.io/<ert-github-användarnamn>/template-app-backend:latest
    restart: unless-stopped

  frontend:
    image: ghcr.io/<ert-github-användarnamn>/template-app-frontend:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    depends_on:
      - backend
EOF
```

Byt ut `<ert-github-användarnamn>` mot GitHub-namnet på den som äger
repot (gemener — GHCR kräver det, precis som i M3/M6). Rör inget annat i
filen: tjänstenamnet `backend` måste stå kvar — `nginx.conf` i
frontend-imagen proxar till `backend:8000`, precis som i M3 — och
`8080:8080` är samma mapping som i M3. `:latest` räcker idag; sha-taggen
från M6 steg 1 är den ni skulle välja för att köra exakt en viss commit.

```bash
sudo docker compose -f /opt/app/docker-compose.yml pull
sudo docker compose -f /opt/app/docker-compose.yml up -d
```

**Om `pull` svarar `denied`:** exakt samma synlighetsfälla som i M3 och
M6 — ett GHCR-paket är **privat** som standard. Gå tillbaka till er
repos **Packages**-flik på GitHub, öppna paketet → **Package settings**
→ **Change visibility** → **Public**, upprepa för båda paketen, och kör
`pull` igen. Det är precis den kontroll M6 steg 4 redan bad er göra —
om ni missade den då fastnar ni här.

## Steg 10 – Verifiera från VM:en, sen utifrån

På VM:en:

```bash
sudo docker compose -f /opt/app/docker-compose.yml ps
curl http://localhost:8080/api/health
```

`ps` ska visa båda tjänsterna (`backend`, `frontend`) med **STATUS** `Up`,
och `curl` ska svara `{"status":"ok"}`. Svarar den `Connection refused`
hann containrarna inte starta — vänta några sekunder och kör om.

Logga sedan **ut** ur SSH-sessionen (`exit`) och kör samma anrop
**utifrån** — från er codespace eller er egen dator. Poängen är att
anropet kommer utanför CSC:s nät och måste gå via floating IP:n och
security groupen; codespacen ligger lika mycket utanför som er egen
dator, så den duger som bevis:

```bash
curl http://<er-floating-ip>:8080/api/health
```

Svaret ska vara samma `{"status":"ok"}`. Fungerar det: prova
nip.io-URL:en också. Byt punkterna i floating-IP:n
mot bindestreck:

```bash
curl http://<ip-med-bindestreck>.nip.io:8080/api/health
```

Exempel: `195.148.X.Y` → `http://195-148-X-Y.nip.io:8080/api/health`.
Öppna `http://195-148-X-Y.nip.io:8080/` (roten, utan `/api/health`) i en
webbläsare — nu ska ni se själva notes-appen. (Provar ni `/api/health` i
webbläsaren ser ni bara JSON-svaret — det är väntat.)

> 📸 **Kom ihåg skärmdump till inlämningen:** terminalen i codespacen eller på er dator med `curl` mot floating-IP:n **utifrån** och svaret `{"status":"ok"}`.

> 📸 **Kom ihåg skärmdump till inlämningen:** webbläsaren med notes-appen på nip.io-URL:en, adressfältet synligt.

## Steg 11 – Lämna in beviset och tagga milstolpen

**Bevisa det osynliga:** i den här milstolpen syns nästan inget arbete i
repot — allt hände i Horizon-konsolen och på VM:en. Ni har nu 5
skärmdumpar från steg 2, 3, 5 och 10 — lägg dem i `inlamning/` och
skriv några meningar per skärmdump i `inlamning/m7-cloud.md`:
konsolstegen (key pair, security group, instans, floating IP),
SSH-sessionen, `docker-compose.yml`-filen på VM:en, `curl`-beviset
utifrån. Formatet visas i
`inlamning/m0-exempel.md`. (Skärmdumparna hamnar på er egen dator, inte i
codespacen — dra in filerna i codespacens filutforskare, eller använd
**Upload**.) Detta är ert eget bevis och ersätter inte lärarens kontroll
av konsolstegen ovan.

Logga ut från VM:en (`exit`) och stå i **ert eget repo**, inte i
`course-material` (där steg 0 lämnade er). I codespacen ligger ert repo i
`/workspaces/` (lokalt: `cd` till er klon). `main` är skyddad sedan M2, så
även beviset går in via en PR:

```bash
cd /workspaces/<ditt-repo>

git switch main && git pull
git switch -c m7-inlamning
git add inlamning/
git commit -m "docs: add M7 proof of the manually built cloud VM"
git push -u origin m7-inlamning
```

Rör inget annat i repot — PR:en innehåller bara `inlamning/`
(`docker-compose.yml` på VM:en hör inte hemma i repot).

Öppna pull requesten. Beskrivningen säger vad, och **"Så testar du"**:
`curl http://<er-floating-ip>:8080/api/health` ska svara `{"status":"ok"}`
och nip.io-URL:en ska visa appen. **Par:** buddyn kör raden, läser filen
och godkänner (**Review changes** → **Approve**) — egen approve räknas
aldrig. **Solo:** självgranskning som i M2 (minst en egen radkommentar i
`inlamning/m7-cloud.md`). Vänta på grön check, merga, **Delete branch**.

Efter mergen visar **Actions** också en röd `Deploy to VM` — samma väntade
röda som sedan M1 steg 4. Den blir **inte** grön av att VM:en nu finns:
deploy-nyckeln, deploy-användaren och secrets saknas fortfarande, och de
kommer i M9.

**Kontrollera på GitHub** att `inlamning/m7-cloud.md` ligger på `main` —
tagga då:

```bash
git switch main && git pull
git tag m7-cloud
git push origin m7-cloud
```

**AI-verktyg:** samma regel som i M2, M5 och M6 — Claude Code, Codex och
Gemini i devcontainern får hjälpa er med kommandona och
`docker-compose.yml`-syntaxen. Meningarna i `inlamning/m7-cloud.md`
skriver ni själva, med egna ord.
Och klistra aldrig in en privat nyckel i ett AI-verktyg.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] VM:en syns som **Active** i **Compute → Instances**.
- [ ] Security groupen tillåter Ingress på port 22 och 8080.
- [ ] En floating IP är associerad med instansen.
- [ ] `ssh -i ~/.ssh/cpouta_ed25519 ubuntu@<floating-ip>` loggar in utan fel
      från er egen arbetsmiljö (codespace eller egen dator) med er egen
      nyckel. **I par:** för var och en av er — bådas publika nycklar
      ligger i `~/.ssh/authorized_keys` på VM:en.
- [ ] `sudo docker compose -f /opt/app/docker-compose.yml ps` på VM:en
      visar båda tjänsterna med STATUS `Up`.
- [ ] `curl http://<floating-ip>:8080/api/health` svarar `{"status":"ok"}`
      **utifrån** — från er codespace eller er egen dator, inte bara från
      VM:en via SSH.
- [ ] nip.io-URL:en visar appen i en vanlig webbläsare.
- [ ] Det osynliga arbetet är dokumenterat med 5 skärmdumpar i
      `inlamning/m7-cloud.md`, mergat till `main` via en egen granskad PR
      — godkänd av buddyn, eller med radkommentar (solo); branchen
      `m7-inlamning` är raderad.
- [ ] Taggen `m7-cloud` pekar på `main` efter sista mergen och syns
      under **Tags**.

## Vanliga problem

**SSH svarar `Connection timed out` eller `Connection refused`.**
Nästan alltid security group-regeln — kontrollera att SSH-regeln (port
22) verkligen är **Ingress**, inte **Egress** (Egress är redan öppet som
standard och löser inget här). Kontrollera också att floating IP:n
faktiskt är associerad (steg 5) — utan den svarar ingenting alls utifrån.

**SSH svarar `Permissions 0644 for '~/.ssh/cpouta_ed25519' are too open`.**
Er privata nyckel har fel filrättigheter. Kör `chmod 600
~/.ssh/cpouta_ed25519` och försök igen.

**SSH svarar `Permission denied (publickey)`.** Fel nyckel eller fel
användare: logga in som `ubuntu` (inte `root`) med `-i
~/.ssh/cpouta_ed25519`, och kontrollera att det är den nyckel VM:en
startades med (steg 4) — eller, för den andra i paret, att er publika
nyckel faktiskt hamnade i `authorized_keys` på VM:en (steg 7).

**`ssh-add` svarar `Could not open a connection to your authentication
agent`.** Codespacen kör ingen agent — starta en enligt steg 1
(`eval "$(ssh-agent -s)"`), eller hoppa över agenten och kör `ssh -i`.

**Docker svarar `permission denied while trying to connect to the Docker
daemon socket`.** Ni glömde `sudo`. Användaren `ubuntu` är inte med i
`docker`-gruppen, och alla Docker-kommandon i labben körs med `sudo`.

**`docker compose` svarar `no configuration file provided: not found`.**
Ni står inte i `/opt/app` och glömde `-f /opt/app/docker-compose.yml`.

**`docker pull`/`docker compose pull` svarar `denied: requested access
to the resource is denied`.** Ett GHCR-paket är fortfarande **privat**
— se steg 9, gör båda paketen publika via **Package settings → Change
visibility → Public** och kör `pull` igen. Samma fälla som i M3 och M6,
nu med en molnserver som motpart i stället för er egen arbetsmiljö.

**`docker compose pull` svarar `invalid reference format: repository name
must be lowercase`.** GitHub-namnet i `/opt/app/docker-compose.yml` har
versaler — skriv om det med små bokstäver (samma regel som i M3 och M6)
och kör `pull` igen.

**`curl` svarar `Connection refused` direkt efter `up -d`.** Containrarna
hade inte hunnit starta — vänta några sekunder och kör om. Svarar den
fortfarande inte: `sudo docker compose -f /opt/app/docker-compose.yml
logs`.

**`exec format error` i loggarna, en container startar om hela tiden.**
Imagen byggdes för arm64 (för hand på en Mac med Apple-kisel i M3) men
VM:en är amd64 — skillnaden M3:s Vanliga problem varnade för. Kör **Run
workflow** på `Publish images` med **`main`** vald (M6 steg 3 — en körning
på en annan branch pushar inget) så att CI bygger om imagen för amd64,
och kör sedan `pull` + `up -d` igen.

**`curl` mot floating-IP:n fungerar inte från min codespace/dator, men
fungerar på VM:en (`localhost`).** Antingen är floating IP:n inte associerad
(steg 5), eller så saknar security groupen fortfarande port 8080 som
Ingress (steg 3). `localhost` på VM:en går förbi hela security-group-
lagret, så det testar inte samma sak.

**Instansen startade men den har bara en `192.168.X.Y`-adress.** Ingen
floating IP är associerad ännu. Står det **No floating IP addresses
allocated** i dialogen har projektet ingen ledig IP — allokera en med
**+**-knappen i steg 5 innan ni kan associera. Kvoten är begränsad per
projekt, och ert projekt är litet: har ni redan allokerat IP:er ni inte
använder, släpp dem (**Network → Floating IPs → Release Floating IP**) i stället
för att be om mer kvot. Räcker kvoten ändå inte, fråga läraren.

**Checken `Lint and test backend` är röd på inlämnings-PR:en fast ni bara
lagt till filer i `inlamning/`.** Då var `main` redan röd innan ni började
— samma diagnos som M5:s Vanliga problem: läs loggen under **Details**,
fixa det den pekar på i samma PR, pusha.

**Min PR kan inte mergas trots grön check (Required approvals).** Samma
regel som sedan M2: är ni ett par kräver rulesetet en godkänd review —
be er buddy titta. Solo: *Required approvals* är 0, så självgranska
PR:en som i M2 och merga. Står den på 1 för dig som jobbar solo — sätt
tillbaka den till 0 (M2 steg 1).

**`Deploy to VM` är röd i Actions fast VM:en nu finns.** Väntat — den
blir grön först i M9, när deploy-nyckeln, deploy-användaren och secrets
finns. Den är ingen check på er PR och stoppar ingen merge.

**Taggen pekar på fel commit / syns inte på GitHub.** Samma recept som
M1:s Vanliga problem: `git tag -d m7-cloud`, `git push --delete origin
m7-cloud`, `git switch main && git pull`, tagga om, `git push origin
m7-cloud`. **I par:** har par-kompisen redan hämtat den gamla taggen
behöver hen `git fetch --tags --force`.

## Källor

- <https://docs.csc.fi/cloud/pouta/launch-vm-from-web-gui/>
- <https://docs.csc.fi/cloud/pouta/networking/>
- <https://docs.csc.fi/cloud/pouta/tutorials/ssh-key/>
