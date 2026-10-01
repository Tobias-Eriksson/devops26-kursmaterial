# M8 – Terraform återskapar M7, riv & bygg om, samma floating IP

Den här labben hör till **session 8** (Infrastructure as Code). Ni jobbar
i par (eller solo), ca 90 minuter.

**Milstolpens bevis:** `terraform apply` från noll ger en fungerande
miljö (samma resultat som M7 — appen svarar på floating-IP/nip.io-URL:en),
`terraform state list` visar alla resurser Terraform bokför, rebuild-demot
(riv VM:en, bygg om) bevisar att floating IP:n är **exakt densamma** före
och efter, M7:s handbyggda VM och dess floating IP är borta, 6 skärmdumpar
finns i `inlamning/m8-iac.md`, mergat via en egen PR, och taggen `m8-iac`
är uppe.

**Viktigt att veta innan ni börjar:** modulen ni kör idag ligger i
`iac/terraform/` i template-repot (samma repo som resten av kursen) och
är INTE ett leksaksexempel — det är den riktiga koden, redan granskad.
Er uppgift idag är att köra den mot ert eget CSC-projekt, inte att skriva
Terraform från grunden.

**Om kommandona nedan:** de flesta stegen kräver riktig åtkomst till CSC
cPouta och är märkta **[CLOUD]** i rubriken. Läraren körde kedjan i
cPouta 2026-08-13 och skärmdumparna i stegen kommer från den körningen —
modulen fick sin port-resurs samma eftermiddag, därför visar två av
bilderna (steg 4 och 9) sju resurser där ni ser åtta.

## Steg 0 – Förkrav

- [ ] Uppdatera kursmaterialet: `cd course-material && git pull`.
- [ ] Öppna er codespace (eller er lokala klon) — labben körs i en
      terminal där. **Terraform-staten hamnar bara i den codespacen**
      (`iac/terraform/terraform.tfstate`, aldrig i git): radera eller byt
      inte codespace förrän efter M9, då staten behövs igen.
- [ ] M1–M3 och M5–M7 är klara: taggarna `m1-repo`, `m2-review`,
      `m3-container`, `m5-ci`, `m6-cd-images` och `m7-cloud` är uppe.
      Båda GHCR-paketen (`template-app-backend`, `template-app-frontend`)
      är **Public** (`labs/m6-cd-images.md`, steg 4) — VM:ens cloud-init
      pullar `:latest` **utan inloggning**, och ett privat paket ger
      `denied` helt tyst inne på VM:en.
- [ ] M4 krävs inte för dagens steg — men gör klart det innan session 10.
- [ ] Rulesetet från M2 + M5 är på: **Enforcement status: Active**,
      **Require a pull request before merging**, *Required approvals*
      **1 om ni är ett par, 0 om du jobbar solo**, och **Require status
      checks to pass** med `Lint and test backend`. Dagens inlämnings-PR
      går igenom exakt samma spärr.
- [ ] Checken `Lint and test backend` var **grön** på er senaste mergade
      pull request. Var den **röd**? Då är `main` redan röd och ingen PR
      i dag kan mergas — se Vanliga problem i `labs/m5-ci.md` först.
- [ ] Båda i paret är medlemmar i ert **CSC-projekt** via Haka
      (samma projekt som i M7).
- [ ] `~/.ssh/cpouta_ed25519.pub` från M7 steg 1 finns i den codespace
      (eller på den dator) där ni kör Terraform — `ls ~/.ssh/`. Modulen
      laddar upp den som VM:ens keypair. Saknas den (ny eller ombyggd
      codespace): skapa om nyckelparet enligt M7 steg 1.
- [ ] `terraform` är installerat (`terraform version`) — finns
      förinstallerat i er codespace, kör ni lokalt: installera från
      <https://developer.hashicorp.com/terraform/install> (codespacen
      kör pinnad version 1.15.8).
- [ ] `openstack`-CLI:t är installerat (`openstack --version`) — finns
      förinstallerat i er codespace, kör ni lokalt: `pip install
      python-openstackclient`. Ingen lust att installera något alls?
      Horizon-webbgränssnittet visar samma information, se steg 2 —
      exakta menytexter kan skilja sig mot vad ni ser där.
- [ ] `iac/terraform/` finns **redan** i ert repo — mappen följde med från
      template-repot i M1, precis som workflow-filerna. Ni skriver ingen
      Terraform idag, ni kör den.
- [ ] M7:s VM står kvar och svarar (S7 bad er låta den stå). Steg 6 river
      den. Har ni ingen M7-VM går labben ändå — hoppa då över steg 6.
- [ ] **Par:** båda gör steg 1 (var sin application credential). Sedan
      kör **en** av er steg 2–5 och 7–9 i sin codespace — staten finns
      bara där, och projektets floating-IP-kvot (2) räcker inte för två
      `apply` bredvid M7:s adress. Den andra läser planen högt i steg 4,
      river M7 i Horizon (steg 6) och öppnar inlämnings-PR:en i steg 10 —
      den första granskar och godkänner den.
- [ ] **Solo:** *Required approvals* står kvar på **0** — du granskar din
      PR som i M2 och M5 (beskrivning + minst en egen radkommentar).

## Steg 1 – [CLOUD] clouds.yaml: autentisering mot cPouta

**[CLOUD]**

Terraform loggar inte in som ni — det använder en **application
credential**, en egen nyckel för maskinen med bara de rättigheter den
behöver. Ert CSC-lösenord ska ALDRIG skrivas in i någon fil: en
application credential går att återkalla direkt och har ett utgångsdatum.

1. Logga in på <https://pouta.csc.fi> med Haka, välj ert projekt.
2. **Identity → Application Credentials** (vänstermenyn) → **Create
   Application Credential**.
3. Fyll i dialogen så här:
   - **Name**: t.ex. `devops-m8` — bara en etikett åt er själva.
   - **Secret**: lämna **tomt**. Då genereras en hemlighet åt er, och
     den visas **en enda gång** (nästa punkt).
   - **Expiration Date**: sätt ett datum **efter kursens slut**, t.ex.
     `08.11.2026`. Utgången räknas i **UTC**, och ett datum utan
     klockslag betyder `00:00:00` — sätter ni `31.10.2026` dör
     credentialen redan på morgonen den 31:a, mitt i ert arbete.
   - **Roles**: kryssa i **member** explicit. Det räcker för allt
     Terraform gör i den här labben (instanser, security groups,
     floating IP, nätverk). Väljer ni ingenting alls får credentialen
     **alla** era roller — bredare åtkomst än den behöver.
   - **Access Rules**: lämna tomt. **Unrestricted (dangerous)**:
     lämna okryssat.
4. Klicka **Create Application Credential**. Nu visas dialogen **"Your
   application credential"** med en orange varningsruta: hemligheten går
   inte att läsa ut igen efter att sidan stängts. Klicka
   **DOWNLOAD CLOUDS.YAML** — den nedladdade filen har hemligheten
   ifylld åt er.

   ![Dialogen "Your application credential" i Horizon: fälten ID, Name
   och Secret till vänster, en orange varningsruta om att hemligheten
   inte går att läsa ut efter att sidan stängts, och knapparna DOWNLOAD
   OPENRC FILE, DOWNLOAD CLOUDS.YAML och CLOSE
   längst ner](assets/m8-app-cred.png)

5. Lägg filen på **`~/.config/openstack/clouds.yaml`** — alltså i er
   **hemkatalog**. I codespacen är `~` `/home/vscode`, INTE repot
   (`/workspaces/<repo>/`): en `.config/`-mapp i repot hittas aldrig.
   Samma sak lokalt, med er egen `$HOME`. Jobbar ni i codespacen laddas
   filen ner till er egen dator — dra in den i VS Code-fönstret (den
   hamnar då i repot) och flytta den sedan på plats:

   ```bash
   mkdir -p ~/.config/openstack
   mv <den nedladdade filen> ~/.config/openstack/clouds.yaml
   ```

6. Sätt miljövariabeln till namnet på nyckeln under `clouds:` i filen ni
   just laddade ner — för application credentials heter den `openstack`:

   ```bash
   export OS_CLOUD=openstack
   ```

   Öppna filen och titta efter om ni är osäkra: den innehåller
   `auth_type: v3applicationcredential` med `application_credential_id`
   och `application_credential_secret` — inget `password`, inget
   `username`.

**Filen är fortfarande en hemlighet.** Secreten ger åtkomst till hela
ert projekt, så clouds.yaml **committas ALDRIG till git** (och allra
minst i ett publikt repo), och ni tar aldrig skärmdump av dialogen med
Secret eller av filens innehåll. Tappar ni bort secreten går den inte att läsa
ut igen — skapa då en ny application credential. Var och en i paret
skapar sin **egen** credential i parets projekt, precis som ni har egna
SSH-nycklar sedan M7.

Fullständig genomgång: `iac/terraform/README.md`, avsnitt 1.

## Steg 2 – Egna variabler

Stå i **ert eget repo**, inte i `course-material` (där steg 0 lämnade
er). I codespacen ligger ert repo i `/workspaces/` (lokalt: `cd` till er
klon):

```bash
cd /workspaces/<ditt-repo>/iac/terraform
cp terraform.tfvars.example terraform.tfvars
```

Redigera `terraform.tfvars`:

- `network_name` — hämta ert projekts interna nätverksnamn med
  `openstack network list` (kräver att steg 1 är klart), inte "public".
  Utan CLI:t: Horizon visar samma namn, ungefär under **Project →
  Network → Networks** (exakt menytext kan skilja sig mot vad ni ser).
- `ghcr_owner` — samma gemena namn som `.github/workflows/publish-images.yml`
  publicerar era images under: det ni skrev som `<ert-användarnamn>` i
  M3/M6 och i image-sökvägen i M7:s `docker-compose.yml`. **OBS: skriv
  användarnamnet med små bokstäver** även om det har versaler på GitHub —
  annars avvisar Docker image-sökvägen (`repository name must be
  lowercase`, se felsökningen längst ned).
- `instance_name` — sätt till `m8-<förnamn>`, samma namnkonvention som
  M7:s instansnamn. Då ser ni direkt i Horizon vilken VM, security group
  och keypair som är vems, bredvid M7:s handbyggda. Med default-värdet
  (`template-app`) heter allas resurser likadant — inget felmeddelande,
  men omöjligt att se vems som är vems, och lätt att riva fel sak för
  hand.
- `ssh_public_key_path` — lämna default (`~/.ssh/cpouta_ed25519.pub`,
  nyckeln från M7 steg 1). Rör inget annat i filen.

## Steg 3 – Validering utan molnaccess

Innan ni rör riktig infrastruktur — kontrollera att filerna faktiskt är
giltig Terraform:

```bash
terraform fmt -check
terraform init -backend=false
terraform validate
```

Dessa tre kommandon laddar bara ner OpenStack-providern från Terraform
Registry — ingen `OS_CLOUD`, ingen `clouds.yaml`, och absolut ingen
`plan`/`apply` mot riktig cPouta. Ser ni ett fel här är det ett
syntaxfel i era egna filer, inte ett moln-problem. Förväntat svar:
`fmt -check` skriver ingenting, `validate` svarar `Success! The
configuration is valid.`

## Steg 4 – [CLOUD] init, plan, apply mot cPouta

**[CLOUD]**

```bash
terraform init
terraform plan
terraform apply
```

Läs igenom `plan`-utskriften innan ni skriver `yes` vid `apply` — hur
många resurser ska skapas? Jämför mot modulens åtta resursblock: samma
sex begrepp som M7 (keypair, security group + 2 regler, floating IP,
instans) plus två nya för M8 — nätverksporten (den som gör att floating
IP:n går att koppla, och som gör att VM:en kan bytas ut i steg 8 utan att
adressen försvinner) och associationen mellan floating IP och port.
Planen läser dessutom nätverket som en data source
(`data.openstack_networking_network_v2`) — den radas upp separat och
räknas inte in i "to add".

Så här ser en lyckad `apply` ut — räkna själva: åtta `Creating…`-rader,
en för varje resursblock ovan, följt av `Apply complete!` och
outputs-blocket med `floating_ip`:

Bilden nedan är tagen innan porten (`openstack_networking_port_v2.this`)
fanns i modulen och visar därför bara sju resurser och
"Resources: 7 added" — er egen körning ska visa åtta rader och
"Resources: 8 added":

![Slutet av terraform apply-utskriften: sju resurser (keypair,
security group, två security group-regler, floating IP, instans,
association) går från Creating… till Creation complete med varsitt
id, följt av "Apply complete! Resources: 7 added, 0 changed, 0
destroyed" och outputs-blocket med app_url, app_url_nip_io,
floating_ip och ssh_command — bilden är från före port-resursen
tillkom, er körning visar åtta resurser](assets/m8-apply-output.png)

> 📸 **Kom ihåg skärmdump till inlämningen:** slutet av `apply`-utskriften med `Apply complete! Resources: 8 added` och outputs-blocket (`floating_ip`, `app_url_nip_io`).

Cloud-init behöver en minut eller två efter att `apply` är klar innan
appen svarar — det är inte ett fel om `curl` inte funkar direkt. Det ni
gjorde för hand i M7 steg 8–9 (`get.docker.com`, `/opt/app/docker-compose.yml`,
`docker compose up -d`) gör cloud-init nu vid VM:ens första uppstart. Vill
ni se det: `terraform output ssh_command` ger SSH-raden (lägg till
`-i ~/.ssh/cpouta_ed25519` som i M7), och på VM:en visar `sudo docker
compose -f /opt/app/docker-compose.yml ps` samma två tjänster som i M7.

## Steg 5 – Verifiera utifrån

Från er codespace eller er egen dator — precis som i M7 är poängen att
anropet kommer utanför CSC:s nät:

```bash
curl http://<floating_ip från output>:8080/api/health
```

Svaret ska vara `{"status":"ok"}`, precis som i M7. Prova nip.io-URL:en också (samma `app_url_nip_io`-output):

```bash
curl http://<ip-med-bindestreck>.nip.io:8080/api/health
```

Öppna rot-URL:en (utan `/api/health`), t.ex. `http://195-148-X-Y.nip.io:8080/`,
i en webbläsare — ni ska se er notes-app, precis som i M7, men den här
gången byggd från en fil i stället för handklick. (Provar ni
`/api/health` i webbläsaren ser ni bara JSON-svaret — det är väntat.)

> 📸 **Kom ihåg skärmdump till inlämningen:** webbläsaren med notes-appen på M8:s nip.io-URL (adressfältet synligt) — jämför adressen med `app_url_nip_io` i outputs.

## Steg 6 – [KONSOL] Riv M7-VM:en och släpp dess floating IP

M8:s egen miljö svarar nu på sin egen floating IP (steg 5) — M7:s VM och
adress är från och med nu en dubblett, inte en reserv. Att låta båda stå
kvar upptar dubbelt så mycket av ert projekts kvot i onödan — och
floating-IP-kvoten på ett eget CSC-projekt är liten, så nästa gång ni
behöver en adress (M9, spårvalet) kan den vara slut.

Det här är riskfritt att göra nu: M8:s VM och floating IP skapades av
`terraform apply` i steg 4 som en helt egen uppsättning resurser — att
riva M7:s handbyggda VM rör ingenting i er Terraform-state. Adressen som
"återanvänds" i dag är M8:s egen (steg 4 → steg 8); M7:s adress släpps
här, så er gamla nip.io-URL från M7 slutar svara. **Ingen M7-VM?** Hoppa
över steget.

Kom ihåg från S7: i cPouta har alla projektmedlemmar samma fulla
rättigheter — det som skyddar en VM är namngivning och överenskommelser.
Dubbelkolla namnet innan ni bekräftar: ni raderar `m7-<förnamn>`, inte
partnerns `m8-…`.

I Horizon (<https://pouta.csc.fi>):

1. **Compute → Instances** → hitta er M7-instans (namnet ni gav den för
   hand i M7, t.ex. `m7-<förnamn>` — INTE instansen från steg 4, som
   heter exakt det ni satte som `instance_name` i `terraform.tfvars`)
   → pilen bredvid **Create Snapshot** → **Delete Instance** → bekräfta.
2. **Network → Floating IPs** → hitta M7:s adress (den ni använde i
   M7:s `curl`/webbläsare — INTE den nya `floating_ip`-outputen från
   steg 4) → pilen bredvid **Disassociate** → **Release Floating IP** →
   bekräfta.

Så här ser det ut efteråt, från lärarens egen körning (M7-instansen och
dess adress är borta, bara M8 kvar):

![Instances-listan i Horizon med bara instansen m8-topi kvar: Ubuntu-24.04,
standard.small, key pair m8-topi-key, båda adresserna 192.168.1.231 och
195.148.30.54, status Active och Running — M7-instansen syns inte längre
i listan](assets/m8-teardown-instances.png)

![Floating IPs-listan i Horizon med bara adressen 195.148.30.54 kvar,
mappad till m8-topi 192.168.1.231, status Active, och knappen Allocate IP
to Project inte längre gråad — M7:s adress är släppt och
floating-IP-kvoten frigjord](assets/m8-teardown-floating-ips.png)

> 📸 **Kom ihåg skärmdump till inlämningen:** er egen **Instances**-lista efteråt — bara M8-instansen kvar, M7:s borta.

> 📸 **Kom ihåg skärmdump till inlämningen:** er egen **Floating IPs**-lista efteråt — bara M8:s adress kvar. Bilderna ovan visar hur resultatet ska se ut.

Kontrollera att M8 fortfarande svarar (steg 5) efteråt — M7:s
undanrivning påverkar inte er Terraform-miljö alls.

## Steg 7 – Läs state, rör den aldrig för hand

```bash
terraform state list
```

Det här listar varje resurs Terraform bokför för er miljö — jämför
listan mot `main.tf`:s åtta resursblock. Ni ser nio rader: de åtta
resurserna plus en `data.openstack_networking_network_v2.this`-rad för
nätverket som planen läste i steg 4. **Öppna aldrig
`terraform.tfstate` direkt i en editor som normal arbetsgång** — filen
är intern bokföring, inte något ni redigerar; `terraform state
list`/`show` är rätt verktyg för att INSPEKTERA den.

> 📸 **Kom ihåg skärmdump till inlämningen:** terminalen med `terraform state list` och alla nio raderna.

## Steg 8 – [CLOUD] Rebuild-demot: riv VM:en, behåll IP:n

**[CLOUD]**

Notera `floating_ip`-outputen från steg 4 (eller kör `terraform output
floating_ip` igen) INNAN ni river något:

```bash
terraform output floating_ip
```

Så här ser den kommandoraden ut — samma kommando kör ni igen EFTER
`apply` nedan:

![Terminalen i iac/terraform på main, kommandot terraform output
floating_ip med svaret "195.148.30.54"](assets/m8-rebuild-floating-ip-output.png)

`terraform destroy` är på riktigt: kontrollera att ni står i **er**
`iac/terraform` med **er** state innan ni kör.

Riv bara instansen — porten, associationen och floating-IP-resursen
rörs aldrig (samma kommando som `README.md` avsnitt 4):

```bash
terraform destroy -target=openstack_compute_instance_v2.this

terraform apply
```

```bash
terraform output floating_ip
```

Bilden ovan visar bara formatet på EN körning (lärarens FÖRE-exempel).
Beviset som räknas är ERT eget par: klistra in BÅDA era egna
`floating_ip`-utskrifter, FÖRE och EFTER `destroy`/`apply`, i
`inlamning/m8-iac.md` — de ska vara identiska, ordagrant.

> 📸 **Kom ihåg skärmdump till inlämningen:** terminalen med båda `terraform output floating_ip`-raderna, FÖRE och EFTER `destroy`/`apply` — samma adress.

Verifiera igen med `curl` (steg 5) — appen ska svara på **samma** URL
som innan, trots att VM:en precis byggdes om från noll. Den nya VM:en
fäster vid samma port som förut, så det är inte bara floating IP:n som
överlever — även den interna fixed IP:n kommer tillbaka oförändrad.
Det är hela poängen med att porten är en egen resurs: nätverksidentiteten
ligger kvar i infrastrukturkoden, servern är utbytbar.

## Steg 9 – [CLOUD] Bonus: se vägran med egna ögon

**[CLOUD]** (valfritt, men lärorikt — kommandot
är säkert att köra: det är designat att inte riva något)

```bash
terraform destroy
```

Terraform ska vägra **redan vid plan-steget** med ett fel på
`openstack_networking_floatingip_v2.this` (den har `lifecycle {
prevent_destroy = true }` i `main.tf`). **Ingenting rivs** — inte VM:en,
inte security groupen, inte keypairen, inte associationen. Det är
avsett, inte trasigt.

Så här ser det ut: planen räknar upp alla åtta resurserna under
"to destroy", men körningen stoppas redan där — innan `apply` ens
frågar efter `yes` hinner den aldrig utföras.

Bilden nedan är, precis som i steg 4, tagen innan porten
(`openstack_networking_port_v2.this`) fanns i modulen och visar därför
"Plan: 0 to add, 0 to change, 7 to destroy" — er egen körning ska visa
"8 to destroy", och felet pekar på floating-IP-blockets rad i er
`main.tf` (rad 76, inte 47):

![Terminalen efter terraform destroy: "Plan: 0 to add, 0 to change, 7
to destroy" och Changes to Outputs där app_url, app_url_nip_io,
floating_ip och ssh_command går till null, följt av "Error: Instance
cannot be destroyed" på main.tf rad 47 (resource
openstack_networking_floatingip_v2 "this") med förklaringen att
resursen har lifecycle.prevent_destroy satt men planen kallar på att
den rivs, plus tipset om att antingen stänga av prevent_destroy eller
begränsa planen med -target — bilden är från före port-resursen
tillkom, er körning visar 8 to destroy](assets/m8-destroy-refusal.png)

Kontrollera att appen fortfarande svarar efter detta (steg 5) — inget
ska ha ändrats.

## Steg 10 – Lämna in beviset och tagga milstolpen

**Bevisa det osynliga:** terraform-körningarna lämnar inga spår i repot
(state committas aldrig). Ni har nu 6 skärmdumpar från steg 4, 5, 6, 7
och 8 — lägg dem i `inlamning/` och skriv några meningar per skärmdump i
`inlamning/m8-iac.md`: `apply`-utskriften, appen på M8:s nip.io-URL,
M7-undanrivningen, `terraform state list` och rebuild-beviset att
`floating_ip` är **samma** före och efter. Formatet visas i
`inlamning/m0-exempel.md`. (Skärmdumparna hamnar på er egen dator, inte i
codespacen — dra in filerna i codespacens filutforskare, eller använd
**Upload**.)

Gå tillbaka till repots rot — ni står i `iac/terraform` — innan
git-stegen. `main` är skyddad sedan M2, så även beviset går in via en PR:

```bash
cd /workspaces/<ditt-repo>

git switch main && git pull
git switch -c m8-inlamning
git add inlamning/
git commit -m "docs: add M8 proof of the Terraform-built environment"
git push -u origin m8-inlamning
```

Rör inget annat i repot — PR:en innehåller bara `inlamning/`.
`terraform.tfvars`, `.terraform/`, `terraform.tfstate` och `clouds.yaml`
committas aldrig (`.gitignore` stoppar dem, men kontrollera diffen).

Öppna pull requesten. Beskrivningen säger vad, och **"Så testar du"**:
`curl http://<floating_ip>:8080/api/health` ska svara `{"status":"ok"}`
och nip.io-URL:en ska visa appen. **Par:** buddyn kör raden, läser filen
och godkänner (**Review changes** → **Approve**) — egen approve räknas
aldrig. **Solo:** självgranskning som i M2 (minst en egen radkommentar i
`inlamning/m8-iac.md`). Vänta på grön check, merga, **Delete branch**.

Efter mergen visar **Actions** fortfarande en röd `Deploy to VM` — samma
väntade röda som sedan M1 steg 4. VM:en finns nu, men deploy-nyckeln,
deploy-användaren och secrets saknas fortfarande; de kommer i M9.

**Kontrollera på GitHub** att `inlamning/m8-iac.md` ligger på `main` —
tagga då:

```bash
git switch main && git pull
git tag m8-iac
git push origin m8-iac
```

**AI-verktyg:** samma regel som i M2, M5–M7 — Claude Code, Codex och
Gemini i devcontainern får hjälpa er med kommandona och att läsa
Terraform-utskrifter. Meningarna i `inlamning/m8-iac.md` skriver ni
själva, med egna ord. Och klistra aldrig in `clouds.yaml`, application
credential-secreten eller en privat nyckel i ett AI-verktyg.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] `terraform fmt -check` och `terraform validate` är gröna (steg 3).
- [ ] `terraform apply` från noll gav `Apply complete! Resources: 8 added`.
- [ ] `curl` mot floating-IP:n svarar `{"status":"ok"}` **utifrån** — från
      er codespace eller er egen dator — och nip.io-URL:en visar appen i
      en webbläsare.
- [ ] M7:s VM och floating IP är rivna/släppta i Horizon (steg 6; hade
      ni ingen M7-VM: hoppa över).
- [ ] `terraform state list` visar alla åtta resursblock (plus
      `data.`-raden för nätverket).
- [ ] Rebuild-demot kört: `floating_ip`-outputen är **identisk** före
      och efter.
- [ ] (Bonus) En vanlig `terraform destroy` vägrade vid plan-steget,
      utan att riva något.
- [ ] Det osynliga arbetet är dokumenterat med 6 skärmdumpar i
      `inlamning/m8-iac.md`, mergat till `main` via en egen granskad PR
      — godkänd av buddyn, eller med radkommentar (solo); branchen
      `m8-inlamning` är raderad.
- [ ] Taggen `m8-iac` pekar på `main` efter sista mergen och syns
      under **Tags**.

## Vanliga problem

**`terraform plan`/`apply` säger något om saknad autentisering eller
`Unable to find cloud`.** `OS_CLOUD` är inte satt, eller pekar på fel
namn — kontrollera nyckeln under `clouds:` i er `clouds.yaml` (steg 1;
den nedladdade filen använder `openstack`).

**`Cloud openstack was not found`.** Filen ligger på fel ställe.
Verktygen letar bara i den katalog ni står i, i
`~/.config/openstack/clouds.yaml` och i `/etc/openstack/` — en
`.config/`-mapp i repot hittas aldrig när ni står i `iac/terraform`.
Flytta filen till hemkatalogen (i codespacen `/home/vscode`, inte
`/workspaces/<repo>/`):

```bash
mkdir -p ~/.config/openstack
mv <filen> ~/.config/openstack/clouds.yaml
```

Att lägga den i repot är dessutom farligt: då är det en credential som
kan committas till ett publikt repo.

**Autentiseringsfel efter att ni tappat bort secreten.** Application
credential-hemligheten går inte att läsa ut igen efter att dialogen
stängts — skapa en **ny** application credential (steg 1) och ladda ner
en ny `clouds.yaml`. Ta gärna bort den gamla credentialen i Horizon.

**`apply` klagar på `network_name` eller hittar inget nätverk.**
Ni har antingen inte satt `network_name` i `terraform.tfvars`, eller
gissat ett namn i stället för att hämta det med `openstack network
list` — det är projekt-specifikt, ingen kan gissa det åt er.

**Appen svarar inte trots `Apply complete!`.** Cloud-init tar en minut
eller två efter att instansen startat — vänta och försök igen. Svarar
den fortfarande inte efter 5 minuter: logga in med raden från
`terraform output ssh_command` och kör `sudo docker compose -f
/opt/app/docker-compose.yml logs`.

**Loggen på VM:en säger `denied: requested access to the resource is
denied`.** Samma synlighetsfälla som M3/M6/M7, nu inne i cloud-init: ett
GHCR-paket är fortfarande privat — **Package settings → Change
visibility → Public** (M6 steg 4), kör sedan `sudo docker compose -f
/opt/app/docker-compose.yml pull` och `up -d` på VM:en.

**Loggen säger `invalid reference format: repository name must be
lowercase`.** `ghcr_owner` i `terraform.tfvars` har versaler — skriv om
med små bokstäver (samma regel som i M3/M6/M7) och kör `terraform apply`
igen (VM:en byggs om med rätt compose-fil).

**Rebuild-demot ger en ANNAN `floating_ip` än innan.** Kontrollera att
ni verkligen bara `-target`:ade instansen (steg 8) — om
floating-IP-resursen av misstag revs (t.ex. genom att köra `destroy`
utan `-target`, vilket normalt vägrar, se steg 9) måste en ny adress
allokeras och kan bli en annan.

**`terraform destroy` (steg 9) rev faktiskt något.** Kontrollera att
`lifecycle { prevent_destroy = true }` verkligen finns kvar på
`openstack_networking_floatingip_v2.this` i er `main.tf` — om någon
råkat ta bort blocket försvinner skyddet. Återställ det från
template-repots `main` innan ni försöker igen.

**`terraform plan` säger `no file exists at ".../.ssh/cpouta_ed25519.pub"`.**
Nyckeln från M7 steg 1 finns inte i den här codespacen (ny eller
ombyggd) — skapa om nyckelparet enligt M7 steg 1, eller sätt
`ssh_public_key_path` i `terraform.tfvars` till den fil ni har.

**`apply` failar med `Quota exceeded for resources: ['floatingip']`.**
Projektets floating-IP-kvot är liten (2): M7:s adress + en per `apply`.
Bara en i paret kör `apply` (steg 0), och har ni gamla adresser kvar
släpper ni dem (**Network → Floating IPs → Release Floating IP**) — eller
gör steg 6 först.

**`exec format error` i loggarna på VM:en, en container startar om hela
tiden.** Imagen byggdes för arm64 (för hand på en Mac med Apple-kisel i
M3) men VM:en är amd64 — skillnaden M3:s Vanliga problem varnade för. Kör
**Run workflow** på `Publish images` med `main` vald (M6 steg 3), sedan
`terraform destroy -target=openstack_compute_instance_v2.this` + `apply`
så att cloud-init pullar om.

**`git add inlamning/` svarar `pathspec 'inlamning/' did not match`.** Ni
står kvar i `iac/terraform` — `cd /workspaces/<ditt-repo>` först (steg 10).

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
M1:s Vanliga problem: `git tag -d m8-iac`, `git push --delete origin
m8-iac`, `git switch main && git pull`, tagga om, `git push origin
m8-iac`. **I par:** har par-kompisen redan hämtat den gamla taggen
behöver hen `git fetch --tags --force`.
