# M9 – SSH-deploy från main till produktion, secrets-hygien, hela kedjan

Den här labben hör till **session 9** (CD — hela kedjan). Ni jobbar i
par (eller solo), ca 90 minuter. **Detta är grundprojektets sista
milstolpe** — den som hänger med har grunden klar efter idag (M1–M9 bör
vara klara till session 10).

**Milstolpens bevis:** en egen, synlig ändring är mergad till main och
syns automatiskt på er floating-IP/nip.io-URL utan att ni loggat in på
VM:en för hand under själva deployen — **Deploy to VM** (`deploy.yml`)
körde grönt i Actions EFTER att **Publish images** blivit grön, 4
skärmdumpar finns i `inlamning/m9-cd.md`, mergat via en egen PR, och
taggen `m9-cd` är uppe.

**Viktigt att veta innan ni börjar:** `.github/workflows/deploy.yml`
finns **redan** i ert repo — det följde med från template-repot i M1,
precis som `ci.yml` (M5) och `publish-images.yml` (M6) gjorde. Ingen av
er har rört filen. Det som SAKNAS för att den ska fungera är
per-person/per-VM-specifikt och kan inte ligga färdigt i en mall: en
dedikerad deploy-nyckel, en deploy-användare på just er VM, och
secrets/vars i just ert repo. Det är dagens uppgift.

Det är också förklaringen till den röda **Deploy to VM**-körningen ni
sett i Actions-fliken efter varje push och merge till main sedan M1 (M1
steg 4 och M8 steg 10 lovade båda att den blir grön i dag). Den har
försökt logga in **utan uppgifter**: `DEPLOY_HOST`, `DEPLOY_USER` och
nyckeln var tomma — även efter att er VM fanns (M7/M8). Efter dagens lab
blir den grön, och först då är det rimligt att reagera om den blir röd
igen.

Och appen kör ju redan på VM:en: cloud-init (M8) gjorde `pull` + `up -d`
en gång när VM:en startade. Det som ändras i dag är **vem** som kör de
två kommandona vid **nästa** merge — CI:n, inte ni.

**Om kommandona nedan:** stegen som kräver er riktiga VM i cPouta är
märkta **[CLOUD]** i rubriken. Läraren körde hela kedjan 2026-08-14 och
skärmdumparna i steg 3 och 9 kommer från den körningen.

## Steg 0 – Förkrav

- [ ] Uppdatera kursmaterialet: `cd course-material && git pull`.
- [ ] **Samma codespace som i M8** — Terraform-staten bor bara där
      (`iac/terraform/terraform.tfstate`), och VM:ens adress hämtar ni
      med `cd iac/terraform && terraform output floating_ip` (nip.io-
      adressen: `terraform output app_url_nip_io`). Gå sedan tillbaka
      till repots rot: `cd /workspaces/<ditt-repo>` — labbens kommandon
      körs därifrån.
- [ ] M1–M3 och M5–M8 är klara: taggarna `m1-repo`, `m2-review`,
      `m3-container`, `m5-ci`, `m6-cd-images`, `m7-cloud` och `m8-iac`
      är uppe.
- [ ] M4 krävs inte för dagens steg — men `m4-tests` bör vara uppe till
      session 10 (i morgon); den hårda deadlinen för hela grunden M1–M9
      är ert demopass.
- [ ] Båda GHCR-paketen (`template-app-backend`, `template-app-frontend`)
      är **Public** (`labs/m6-cd-images.md`, steg 4). Deploy-användaren
      kör `docker compose pull` på VM:en **utan inloggning** — ett privat
      paket ger `denied` i Deploy-loggen.
- [ ] **M8:s VM kör.** Rev ni den mellan passen (S8:s hållbarhetsråd):
      kör `export OS_CLOUD=openstack` (M8 steg 1 — variabeln försvinner
      när codespacen startas om) och `terraform apply` igen i
      `iac/terraform`, vänta 1–2 minuter på cloud-init och kontrollera
      `curl http://<floating-ip>:8080/api/health`. VM:en får då en ny
      host-nyckel — se Vanliga problem om SSH i steg 3 varnar.
      M7:s VM revs i M8 steg 6 — dagens lab riktar sig mot M8:s
      Terraform-VM. (Hoppade ni över M8 går M7-VM:en också: Docker och
      `/opt/app/docker-compose.yml` finns på båda.)
- [ ] Ni kan SSH:a in som admin-användaren `ubuntu`. **Observera:** på
      M8:s VM finns bara nyckeln från den codespace där `terraform apply`
      kördes (`ssh_public_key_path`) — partnerns nyckel från M7 steg 7
      försvann med M7-VM:en. Steg 3 görs därför från den codespacen.
- [ ] Rulesetet från M2 + M5 är på: **Enforcement status: Active**,
      **Require a pull request before merging**, *Required approvals*
      **1 om ni är ett par, 0 om du jobbar solo**, och **Require status
      checks to pass** med `Lint and test backend`. Dagens två PR:ar går
      igenom exakt samma spärr.
- [ ] Checken `Lint and test backend` var **grön** på er senaste mergade
      pull request. Var den **röd**? Då är `main` redan röd och ingen PR
      i dag kan mergas — se Vanliga problem i `labs/m5-ci.md` först.
- [ ] **Par:** repots **ägare** gör steg 2, 4, 5 och 6 i sin codespace —
      bara ägaren kan sätta secrets (samma regel som paketen i M6 steg
      4), och den privata nyckeln går aldrig mellan er. Steg 3 gör den
      som körde `terraform apply` i M8 (bara hens nyckel finns på VM:en)
      med ägarens `m9_deploy_key.pub` — den publika raden får skickas i
      chatten. Är ägaren och Terraform-köraren samma person gör hen steg
      2–6. Partnern gör steg 7 och öppnar PR:en med den synliga
      ändringen i steg 8; ägaren granskar. Inlämnings-PR:en i steg 11
      öppnar ägaren, partnern granskar — så har ni båda en egen mergad
      PR i M9.
- [ ] **Solo:** *Required approvals* står kvar på **0** — du granskar
      dina PR:ar som i M2 och M5 (beskrivning + minst en egen
      radkommentar).

## Steg 1 – Läs `deploy.yml`, jämför med `publish-images.yml`

Öppna `.github/workflows/deploy.yml`. Notera fyra saker innan ni rör
något:

1. `on: workflow_run: workflows: ["Publish images"]` — inte
   `push: branches: [main]`. Varför inte en vanlig push-trigger? (Svar:
   en push-trigger skulle köra PARALLELLT med **Publish images** och
   kunna hinna `pull` INNAN den nya imagen finns i GHCR.) Namnet i
   hakparentesen matchar `name: Publish images` på rad 1 i
   `publish-images.yml` — filen ni ändrade i M6 men aldrig döpte om.
2. `if: ${{ github.event.workflow_run.conclusion == 'success' }}` —
   `workflow_run` triggar även om **Publish images** MISSLYCKAS, det här
   villkoret stoppar jobbet i så fall.
3. `permissions: contents: read` — jämför med `publish-images.yml`, som
   har `packages: write`. Varför behöver deploy-workflowen inte skriva
   något? (Svar: den öppnar bara en utgående SSH-anslutning.)
4. `env:` överst i jobbet samlar alla secrets/vars — leta upp namnen
   `DEPLOY_USER`, `DEPLOY_HOST`, `DEPLOY_KNOWN_HOSTS`, `DEPLOY_SSH_KEY`.
   Det är EXAKT de fyra namnen ni sätter upp i steg 6 — stava dem tecken
   för tecken som i filen.

## Steg 2 – Skapa en dedikerad deploy-nyckel

Inte samma nyckel som Terraform/M7 använder för admin-åtkomst — en helt
ny, egen för denna workflow. I repots ägares codespace (par) eller er
egen (solo) — eller lokalt om ni kör så:

```bash
ssh-keygen -t ed25519 -C "m9-deploy" -f ~/.ssh/m9_deploy_key -N ""
```

Det ger er `~/.ssh/m9_deploy_key` (privat, blir `DEPLOY_SSH_KEY`) och
`~/.ssh/m9_deploy_key.pub` (publik, går till VM:en i nästa steg).

## Steg 3 – [CLOUD] Sätt upp deploy-användaren på VM:en

SSH in som admin (`ubuntu`, med nyckeln från M7 steg 1 —
`terraform output ssh_command` ger raden, lägg till `-i` som i M8
steg 4):

```bash
ssh -i ~/.ssh/cpouta_ed25519 ubuntu@<floating-ip>
```

Säger SSH `REMOTE HOST IDENTIFICATION HAS CHANGED`? Det är M8:s
rebuild-demo: samma adress, ny host-nyckel — se Vanliga problem.

På VM:en:

```bash
sudo useradd -m deploy
sudo usermod -aG docker deploy
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
```

Hämta innehållet i den publika nyckeln där den skapades i steg 2
(ägarens codespace eller dator):

```bash
cat ~/.ssh/m9_deploy_key.pub
```

Kopiera hela raden — **par:** ägaren skickar den till den som sitter på
VM:en; den publika delen får gå i chatten, den privata aldrig. Det här
är samma `authorized_keys`-mekanik som M7
steg 7 (partnerns publika nyckel), fast för en annan användare. Tillbaka
på VM:en (fortfarande som `ubuntu`), skriv den till `authorized_keys` med
`sudo tee` (samma manöver som M7 steg 9 — katalogen ägs av root, en
vanlig redirect ger "permission denied") — byt ut
`<innehållet i m9_deploy_key.pub>` mot det ni just kopierade:

```bash
echo '<innehållet i m9_deploy_key.pub>' | sudo tee /home/deploy/.ssh/authorized_keys > /dev/null
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
```

**Ärlig brasklapp:** `docker`-gruppen är i praktiken root-ekvivalent —
den som kan prata med Docker-daemonen kan starta en container med
`-v /:/host` och göra vad den vill på värden. "Least privilege" här
begränsar vad som KRÄVS (ingen sudo, inget lösenord, en enda smal
nyckel) — inte vad ett läckt nyckelinnehav faktiskt ger tillgång till.

Kontrollera med `id deploy` — så här ser det ut på lärarens VM (prompten
visar `m8-topi`: M9 deployar mot M8:s Terraform-VM; era gruppnummer kan
skilja sig från exemplet utan att något är fel):

![Terminalen på VM:en, prompten ubuntu@m8-topi, kommandot id deploy
med svaret uid=1001(deploy) gid=1001(deploy)
groups=1001(deploy),988(docker) — deploy-användaren finns och är
medlem i docker-gruppen](assets/m9-deploy-user.png)

> 📸 **Kom ihåg skärmdump till inlämningen:** `id deploy` på er egen VM,
> med `docker` i grupplistan.

Logga ut från VM:en (`exit`) — resten av labben körs från codespacen.

## Steg 4 – [CLOUD] Testa deploy-nyckeln manuellt, INNAN CI får den

Från den codespace där den privata nyckeln finns (par: ägarens) — SSH
fungerar därifrån, samma klient som i M7. Kör exakt det CI:n kommer
köra, fast bara `ps`:

```bash
ssh -i ~/.ssh/m9_deploy_key deploy@<floating-ip> 'docker compose -f /opt/app/docker-compose.yml ps'
```

Båda containrarna ska stå som `Up` — samma utskrift som M7 steg 10, nu
utan `sudo` (`deploy` är med i `docker`-gruppen, det var inte `ubuntu`).
Det bevisar tre saker på en gång: nyckeln släpper in `deploy`, användaren
får prata med Docker, och compose-filen går att läsa.

Fungerar det INTE här kommer det inte fungera i Actions heller — felsök
nu, inte efter att ni bränt en CI-körning på att gissa.

## Steg 5 – [CLOUD] Pinna host-nyckeln

```bash
ssh-keyscan -H <floating-ip>
```

Spara HELA utskriften (en eller flera rader) — det blir värdet på
`DEPLOY_KNOWN_HOSTS` i nästa steg. Använd **floating IP:n**, inte
nip.io-namnet: `known_hosts` matchas mot exakt den värdsträng `ssh`
anropas med (hashad eller ej), och en rad för IP:n gäller inte för
nip.io-namnet — adressen här och `DEPLOY_HOST` måste vara samma sträng.

## Steg 6 – Secrets och variabler i GitHub

I repot på GitHub: **Settings** → **Secrets and variables** → **Actions**.

**Secrets-fliken → New repository secret:**

| Namn | Värde |
|---|---|
| `DEPLOY_SSH_KEY` | Hela innehållet i `~/.ssh/m9_deploy_key` (den PRIVATA nyckeln, steg 2): `cat ~/.ssh/m9_deploy_key`, kopiera alla rader inklusive `-----BEGIN`/`END` |

**Variables-fliken → New repository variable:**

| Namn | Värde |
|---|---|
| `DEPLOY_USER` | `deploy` |
| `DEPLOY_HOST` | Er floating IP — exakt samma adress som ni körde `ssh-keyscan` mot i steg 5 (inte nip.io-namnet) |
| `DEPLOY_KNOWN_HOSTS` | Utskriften från steg 5 |

Bara `DEPLOY_SSH_KEY` är hemlig — host-nyckeln är offentlig information,
den behöver bara pinnas i förväg, inte gömmas som secret. S6:s
tumregel gäller: *skulle ni bry er om värdet läckte i en offentlig
logg?* Ja → secret.

`DEPLOY_SSH_KEY` är dessutom precis den sortens **långlivade** secret
M6 steg 6 varnade för: den dör inte med jobbet som `GITHUB_TOKEN` gör.
Kör aldrig M6:s `echo`-maskeringsövning med den, och läcker den någon
gång — skapa ett nytt nyckelpar (steg 2–3) och byt secreten; det är
rotation i praktiken.

> 📸 **Kom ihåg skärmdump till inlämningen:** fliken **Variables** med
> `DEPLOY_USER`, `DEPLOY_HOST` och `DEPLOY_KNOWN_HOSTS` (värdena är
> offentliga och får synas) och fliken **Secrets** med `DEPLOY_SSH_KEY`
> (värdet visas aldrig). Två bilder som tillsammans räknas som steg 6:s
> skärmdump.

## Steg 7 – Sanity check utan molnaccess

Stå i ert eget repo (`cd /workspaces/<ditt-repo>`), inte i
`course-material` eller `iac/terraform`. Innan ni litar på att workflowen är rätt
formad — kontrollera att den faktiskt är giltig YAML/Actions-syntax.
Kommandot kör i Docker, så codespacen (Docker finns redan där — M3 steg
0: devcontainern kör docker-in-docker) eller en egen dator med Docker
fungerar båda:

```bash
docker run --rm -v "$(pwd)":/repo -w /repo rhysd/actionlint \
  .github/workflows/deploy.yml
```

Noll findings förväntas — filen är redan granskad och oförändrad. Ser
ni fel här har NÅGON råkat redigera filen, inte ett molnproblem.

## Steg 8 – Egen synlig ändring, mergad till main

Fortfarande i ert eget repo:

```bash
git switch main && git pull
git switch -c m9-visible-change
```

Ändra en synlig text i frontend — t.ex. rubriken (`<h1>`) eller
knapptexten (`Add`) i `frontend/index.html`. Valfri ändring går lika
bra, men håll er till `frontend/` (inte workflows eller infra), så
resten av kedjan beter sig förutsägbart — rör ni `backend/` måste
`ruff` och `pytest` fortfarande gå igenom, annars stannar PR:en på
checken. Committa, pusha, öppna en PR.

Beskrivningen säger vad, och **"Så testar du"**: efter mergen ska
**Publish images** och sedan **Deploy to VM** bli gröna i Actions, och
`curl http://<floating-ip>:8080/` visa den nya texten. **Par:** buddyn
granskar och godkänner (**Review changes** → **Approve**) — egen approve
räknas aldrig. **Solo:** självgranskning som i M2 (minst en egen
radkommentar). Vänta på grön `Lint and test backend` (obligatorisk sedan
M5), merga (**Create a merge commit**), **Delete branch**.

## Steg 9 – [CLOUD] Se kedjan köra i Actions

Öppna **Actions**-fliken. Ni ska se, i ordning:

1. **Publish images** startar direkt vid mergen, bygger och pushar båda
   images till GHCR
2. Först EFTER att den blivit grön: **Deploy to VM** startar via
   `workflow_run`, SSH:ar in och kör
   `docker compose -f /opt/app/docker-compose.yml pull` och sedan
   `up -d` — samma två kommandon som M7 steg 9, nu utan `sudo`

Klicka in på **Deploy to VM**-körningen → jobbet **SSH deploy to cPouta
VM** → stegen **Configure SSH (deploy key + pinned host key)** och
**Deploy (pull + restart stack)** ska båda vara gröna.

Notera vad som INTE händer: ingen av de två workflowarna kör testerna.
De kördes på PR:en i steg 8 som `Lint and test backend` — det är
rulesetet från M5 som hindrar otestad kod från att nå `main`, och därmed
VM:en.

![Actions-fliken, vyn "All workflows", med två gröna körningar i rätt
ordning: PR-mergen till main triggade Publish images #9 (29s), som blev
grön och startade Deploy to VM #9 (20s), också grön — det är kedjan ni
ska känna igen](assets/m9-actions-chain.png)

> 📸 **Kom ihåg skärmdump till inlämningen:** er egen kedja från
> PR-mergen i steg 8 — **Publish images** grön, **Deploy to VM** grön,
> i den ordningen (bilden ovan visar hur den ska se ut).

## Steg 10 – Verifiera utifrån

Från er codespace eller er egen dator — poängen är att anropet kommer
utifrån, inte från VM:en själv:

```bash
curl http://<floating-ip>:8080/
curl http://<floating-ip>:8080/api/health
```

Den första visar er nya text, den andra svarar `{"status":"ok"}` — det
är `/api/health`-svaret M2 bad er låta vara, nu som driftstest. Öppna
sedan nip.io-adressen i en webbläsare, samma form som M7 steg 10:
`http://<ip-med-bindestreck>.nip.io:8080/` — er text-ändring ska synas,
utan att ni loggat in på VM:en för hand under själva deployen (bara i
steg 3, som är engångsuppsättning, inte en del av flödet varje gång).

> 📸 **Kom ihåg skärmdump till inlämningen:** webbläsaren på
> nip.io-URL:en med er ändring synlig — adressfältet med.

## Steg 11 – Lämna in beviset och tagga milstolpen

**Bevisa det osynliga:** deploy-användaren på VM:en, GitHub
Secrets/Variables och själva kedjekörningen syns inte i repot. Ni har nu
4 skärmdumpar från steg 3, 6, 9 och 10 — lägg dem i `inlamning/` och
skriv några meningar per skärmdump i `inlamning/m9-cd.md`. Formatet
visas i `inlamning/m0-exempel.md`. (Skärmdumparna hamnar på er egen
dator, inte i codespacen — dra in filerna i codespacens filutforskare,
eller använd **Upload**.) Committa aldrig själva nyckeln eller
secret-värdena.

`main` är skyddad sedan M2, så även beviset går in via en PR. Stå i
repots rot:

```bash
git switch main && git pull
git switch -c m9-inlamning
git add inlamning/
git commit -m "docs: add M9 proof"
git push -u origin m9-inlamning
```

Rör inget annat i repot — PR:en innehåller bara `inlamning/`.

Öppna pull requesten. **Par:** buddyn läser filen och godkänner
(**Review changes** → **Approve**). **Solo:** minst en egen
radkommentar. Vänta på grön check, merga, **Delete branch**. Mergen kör
kedjan en gång till (**Publish images** triggar på varje push till
`main`) — ofarligt, det är så det ska vara från och med nu; skärmdumpen
i steg 9 är från steg 8:s merge.

**Kontrollera på GitHub** att `inlamning/m9-cd.md` ligger på `main` —
tagga då:

```bash
git switch main && git pull
git tag m9-cd
git push origin m9-cd
```

**Grundprojektet (M1–M9) är nu klart.** Från och med nu: bygg inte om
VM:en. Gör ni det ändå (`terraform destroy -target` + `apply`) är
deploy-användaren borta och host-nyckeln ny — gör om steg 3–5 och byt
`DEPLOY_KNOWN_HOSTS`.

**AI-verktyg:** samma regel som i M2, M5–M8 — Claude Code, Codex och
Gemini i devcontainern får hjälpa er med kommandona och att läsa
Actions-loggar. Meningarna i `inlamning/m9-cd.md` skriver ni själva,
med egna ord. Och klistra **aldrig** in `~/.ssh/m9_deploy_key`,
`DEPLOY_SSH_KEY` eller något annat secret-värde i ett AI-verktyg eller
en chatt.

Vill ni läsa mer om varför modellen ni just byggt är medvetet enkel:
README:ts avsnitt "Vilka är svagheterna här?" (samma sektion
`deploy.yml` hör hemma i) sammanfattar det läraren varnade för före
labben — och vad spår B gör annorlunda.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] `actionlint` mot `deploy.yml` är grönt, noll findings (steg 7).
- [ ] `id deploy` på VM:en visar gruppen `docker` (steg 3).
- [ ] Deploy-nyckeln testad manuellt över SSH mot `deploy`-användaren:
      `docker compose … ps` visar båda containrarna `Up` (steg 4).
- [ ] Secrets/vars satta i GitHub: `DEPLOY_SSH_KEY`, `DEPLOY_USER`,
      `DEPLOY_HOST` (floating IP), `DEPLOY_KNOWN_HOSTS` (steg 6).
- [ ] En egen PR med en synlig ändring är mergad till `main` efter
      review (par: buddyns Approve / solo: egen radkommentar) med grön
      `Lint and test backend`; branchen är raderad (steg 8).
- [ ] **Publish images** och sedan **Deploy to VM** körde gröna i
      Actions, i den ordningen (steg 9).
- [ ] Ändringen syns på er floating-IP/nip.io-URL och `/api/health`
      svarar `{"status":"ok"}` — testat **utifrån**, från er codespace
      eller er egen dator (steg 10).
- [ ] Det osynliga arbetet är dokumenterat med 4 skärmdumpar i
      `inlamning/m9-cd.md`, mergat till `main` via en egen granskad PR
      — godkänd av buddyn, eller med radkommentar (solo); branchen
      `m9-inlamning` är raderad.
- [ ] **I par:** båda har en egen mergad PR i M9 (steg 8 och steg 11).
- [ ] Taggen `m9-cd` pekar på `main` efter sista mergen och syns under
      **Tags**.

## Vanliga problem

**Deploy-jobbet körs aldrig, trots att Publish images blev grönt.**
Kontrollera att `workflows: ["Publish images"]` i `deploy.yml` matchar
`name:`-fältet i `publish-images.yml` EXAKT, tecken för tecken — det är
namnmatchning, inte filnamnsmatchning (samma namnfälla som rulesetet i
M5). Startade ni **Publish images** för hand med **Run workflow** på en
annan branch än `main`? Då startar Deploy aldrig — `branches: [main]`
i triggern.

**Deploy to VM är röd på steget `Deploy (pull + restart stack)` med
`Host key verification failed.`** Tre orsaker, i sannolikhetsordning:
`DEPLOY_HOST` är nip.io-namnet men ni skannade IP:n (eller tvärtom) —
de måste vara exakt samma sträng; `DEPLOY_KNOWN_HOSTS` är ofullständigt
kopierad (måste vara HELA `ssh-keyscan`-utskriften); eller VM:en har
byggts om sedan steg 5 (ny host-nyckel på samma adress) — kör steg 5
igen och ersätt variabeln.

**`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` när ni SSH:ar från
codespacen.** M8:s rebuild-demo gav VM:en en ny host-nyckel på samma
adress. Ta bort den gamla raden med `ssh-keygen -R <floating-ip>`,
anslut igen och svara `yes`.

**`Permission denied (publickey)` för `deploy`** (i steg 4 eller i
Deploy-loggen). Kontrollera på VM:en att `/home/deploy/.ssh/authorized_keys`
innehåller hela raden från `m9_deploy_key.pub`, är `600` och ägs av
`deploy` (steg 3), och att `DEPLOY_SSH_KEY` är hela den PRIVATA nyckeln
inklusive `-----BEGIN`/`END`-raderna. Byggde ni om VM:en efter steg 3
finns användaren inte längre — cloud-init skapar den inte (första felet
är då `Host key verification failed`, se ovan) — gör om steg 3–5 och
byt `DEPLOY_KNOWN_HOSTS` (steg 6).

**`docker compose` på VM:en ger `permission denied while trying to
connect to the docker API at unix:///var/run/docker.sock`** (äldre
Docker: `…the Docker daemon socket…`).
Deploy-användaren saknar `docker`-gruppen — kör
`sudo usermod -aG docker deploy` igen och testa steg 4 på nytt (ny
SSH-session krävs för att gruppmedlemskapet ska slå igenom).

**Deploy-loggen säger `error from registry: denied` på `pull`.** Samma
synlighetsfälla som M3/M6/M7/M8: ett GHCR-paket är fortfarande privat —
**Package settings → Change visibility → Public** (M6 steg 4), båda
paketen. Kör sedan om jobbet (**Re-run all jobs**).

**Actions-jobbet timear ut på SSH, fast steg 4 fungerade från
codespacen.** Satte ni `ssh_allowed_cidr` i `terraform.tfvars` till en
enda adress (t.ex. er egen `/32`)? Då släpps GitHubs runners inte in —
de har varierande adresser. Ta bort raden (standard `0.0.0.0/0`, samma
som M7 steg 3) och kör `terraform apply` i `iac/terraform` — bara
security group-regeln ändras, inte VM:en.

**Jag har rättat secrets/variablerna — hur kör jag om utan ny commit?**
**Re-run all jobs** på den röda **Deploy to VM**-körningen (den läser
secrets på nytt). Eller **Actions → Publish images → Run workflow** på
`main` (M6:s `workflow_dispatch`) — då kör hela kedjan en gång till.

**Ändringen syns inte trots grönt i Actions.** Webbläsarcache — testa
`curl` istället för webbläsaren (steg 10), eller hård-ladda om sidan.
Direkt efter deployen kan `/api/` svara `502` i några sekunder
(`up -d` väntar inte på att backend är redo) — vänta och försök igen.
Fortfarande inget: kontrollera att PR:en verkligen mergades till `main`
och inte bara stängdes, och kör steg 4:s rad med `ps` — kolumnen
`CREATED` ska vara färsk; annars `… logs frontend` på samma sätt.

**PR:en står på "Expected — Waiting for status to be reported" och går
inte att merga.** Någon kryssade i `SSH deploy to cPouta VM` eller `Build
and push …` som obligatorisk check i rulesetet (varningen i M5 steg 1) —
de körs aldrig på PR:ar. Ta bort dem under **Settings → Rulesets**; bara
`Lint and test backend` ska stå där.

**Checken `Lint and test backend` är röd på PR:en.** Då var `main` redan
röd innan ni började (samma diagnos som M5:s Vanliga problem), eller så
ändrade ni i `backend/` och ett test gick sönder — läs loggen under
**Details**, fixa i samma PR, pusha.

**Min PR kan inte mergas trots grön check (Required approvals).** Samma
regel som sedan M2: är ni ett par kräver rulesetet en godkänd review —
be er buddy titta. Solo: *Required approvals* är 0, så självgranska
PR:en som i M2 och merga. Står den på 1 för dig som jobbar solo — sätt
tillbaka den till 0 (M2 steg 1).

**Min körning står och väntar ("Queued") i flera minuter.** Inget fel —
GitHub-hostade runners kan ta en stund att tilldelas. Kedjan kör i tur
och ordning; vänta.

**`actionlint` (steg 7) hittar fel i `deploy.yml`.** Filen ska vara
oförändrad sedan M1 — någon har troligen redigerat den av misstag.
Taggen `m1-repo` har den orörda filen:

```bash
git checkout m1-repo -- .github/workflows/deploy.yml
```

Committa återställningen via en PR innan ni går vidare.

**Gamla röda Deploy to VM-körningar ligger kvar i listan.** Väntat — de
försvinner inte. Den första gröna kommer vid nästa merge till `main`
(steg 8), eller via **Re-run all jobs**.

**Taggen pekar på fel commit / syns inte på GitHub.** Samma recept som
M1:s Vanliga problem: `git tag -d m9-cd`, `git push --delete origin
m9-cd`, `git switch main && git pull`, tagga om, `git push origin
m9-cd`. **I par:** har par-kompisen redan hämtat den gamla taggen
behöver hen `git fetch --tags --force`.
