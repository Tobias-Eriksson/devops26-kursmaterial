# M1 – Konton, eget repo, första commit

Den här labben hör till **session 1** (Vad är DevOps?) — steg 0
(kontocheck) tar mest tid första gången du gör den.

**Milstolpens bevis:** ert repo finns på GitHub, skapat från kursens
template, med minst en egen commit pushad, taggen `m1-repo` uppe, läraren
inbjuden som collaborator och GitHub-länken inlämnad på itslearning under
**"Milstolpe 1"**.

Repot ni skapar idag är **samma repo ni jobbar i hela kursen** — det ni
demo:ar på demodagen i oktober är en direkt fortsättning på det ni gör nu.
Behandla det därefter.

## Steg 0 – Kontokoll

Normalfallet: det här är första gången du gör kontochecken — ingen
förberedelse hemma förväntas. (Samma innehåll finns i
`onboarding/student-onboarding.md`, om du vill ha den på ett separat blad.)

- [ ] Jag kan logga in på GitHub.
- **Valfritt:** ansök om **GitHub Education** (fler gratis Actions-minuter):
      https://education.github.com/discount_requests/application — *inte
      nödvändigt*, publika repon har redan fria Actions-minuter och
      Codespaces gratisnivå räcker för kursen.
- [ ] Jag har ett CSC-konto via **Haka** (https://sui.csc.fi — logga in med
      din Arcada-inloggning) och kan logga in på **MyCSC**
      (https://my.csc.fi) med samma inloggning.

Kontot är allt som behövs idag. **Ert eget CSC-projekt** (det som ger er
tillgång till molnet) skapar ni efter labben — se avsnittet **"Efter
labben"**, deadline före session 7.

**CSC strular?** Det är *inte* blockerande idag — molnet behövs på riktigt
först i session 7. Säg till läraren så det inte glöms bort, och fortsätt
med steg 1.

## Steg 1 – Solo eller par?

Par rekommenderas (och bildas nu). Regler för par:

- **Ett** repo per par. Den ena skapar det i steg 2 och bjuder in den andra
  som **collaborator** (repo → *Settings* → *Collaborators* → *Add people*).
- **Båda** gör minst en egen commit redan idag, under **eget** GitHub-konto.
  Betyget är individuellt och Git-historiken är betygsunderlag — från dag 1.

## Steg 2 – Skapa ert repo från template

1. Öppna kursens template-repo: `https://github.com/Tobias-Eriksson/devops26-template`.
2. Klicka **Use this template** → **Create a new repository**.
   *Inte* Fork — en fork pekar tillbaka på originalet, ett template-repo
   ger er en egen fristående historia.
3. Owner: ditt eget konto. Namn: valfritt vettigt, t.ex. `devops-notes`.
4. Synlighet: **Public** krävs för kursens flöde — obegränsade gratis
   Actions-minuter, ni kan visa upp repot efteråt, och (viktigast) branch
   protection och rulesets **enforce:as inte alls** på Private-repon på
   ett gratis GitHub-konto: M2:s skydd och M5:s break-the-build-övning
   blir verkningslösa utan att något syns. **Private funkar bara** om ni
   har **GitHub Pro** eller ett aktiverat **GitHub Education**-paket
   (steg 0 ovan) — annars, välj Public.
5. Bjud in läraren som **collaborator** — alltid, oavsett synlighet (det
   behövs för betygsättningen): repo → **Settings** → **Collaborators** →
   **Add people** → användarnamn `Tobias-Eriksson`.

**Tips, medan ni ändå är inne i inställningarna:** GitHub mejlar er för varje
misslyckad Actions-körning (workflows kör automatiskt från M1) — vill ni
slippa det, gå till **github.com/settings/notifications** → sektionen
**Actions** → bocka ur **Email** (webbnotiser i Actions-fliken finns kvar).

## Steg 3 – Öppna utvecklingsmiljön

**Alternativ A – Codespaces (rekommenderas, bara webbläsare):**
på repots GitHub-sida: **Code** → **Codespaces** → **Create codespace on
main**. Efter någon minut har du VS Code i webbläsaren med kursens alla
verktyg förinstallerade (repots devcontainer). Git är redan inloggat som du.
GitHub raderar en codespace efter 30 dagars inaktivitet — hoppa in minst en
gång i månaden så håller samma codespace hela kursen. Inget repo-innehåll
försvinner (allt pushat ligger kvar på GitHub), men miljön och ev.
ocommittade ändringar är borta, och devcontainern byggs om från grunden.

**Alternativ B – lokalt:** kräver Git (och för att köra appen: Docker).

```bash
git clone https://github.com/<ditt-användarnamn>/<ditt-repo>.git
cd <ditt-repo>
```

**Kursmaterialet:** devcontainern klonar automatiskt in kursens material
som mappen `course-material/` i repot — inget ni gör själva. Mappen är
listad i `.gitignore`: den syns aldrig i `git status` eller i era commits.
Materialet uppdateras varje vecka, så innan varje lab: `cd course-material && git pull`.
Kör ni lokalt utan devcontainer: klona själva en gång med
`git clone https://github.com/Tobias-Eriksson/devops26-kursmaterial.git course-material`
i repo-roten — sedan fungerar samma `git pull`.

## Steg 4 – Första committen

Gör en riktig ändring med innehåll, inte en tom testcommit: öppna
`README.md` och lägg till en rad under rubriken med vilka ni är, t.ex.
`Team: Alice Andersson & Bob Berg`.

Kontrollera först att Git vet vem du är — **det här namnet hamnar i
betygsunderlaget** (i Codespaces är det redan rätt, lokalt kan det saknas):

```bash
git config user.name    # ditt namn
git config user.email   # din e-post (den GitHub-kopplade)
```

Tomt eller fel? Sätt rätt värden:

```bash
git config --global user.name "Alice Andersson"
git config --global user.email "alice@example.com"
```

Committa och pusha:

```bash
git status                          # se vad som ändrats
git add README.md
git commit -m "Add team names to README"
git push
```

Ladda om repo-sidan på GitHub — din ändring och ditt användarnamn ska synas
på committen. **I par:** nu gör den andra sin commit (ändra en rad till i
README, t.ex. era spårönskemål) från sitt eget konto.

**Den röda körningen i Actions-fliken är väntad.** Klicka på **Actions**:
er push startade `Publish images` (grön — den bygger appens images) och
därefter `Deploy to VM`, som blir **röd**. Den försöker logga in på en
server ni bygger först i M9 och saknar sina uppgifter fram till dess; den
är ingen check på era pull requests och blockerar ingenting. Kursens regel
är inte "ignorera rött" utan "ingen röd körning utan känd orsak" — den här
har en, och i M9 blir den grön.

## Steg 5 – Tagga milstolpen

Ni har redan kontrollerat på GitHub att rätt commit ligger uppe (steg 4)
— då, och inte förr, sätter ni taggen. `git tag` fastnar på den commit som
är HEAD i det ögonblick kommandot körs, så en tagg satt för tidigt pekar
inte på den commit ni menade:

```bash
git tag m1-repo
git push origin m1-repo
```

Observera att taggen pushas **separat** — en vanlig `git push` skickar inte
taggar. Ordningen mellan de två pusharna (kod och tagg) spelar ingen roll,
men ordningen committa → pusha → kontrollera på GitHub → tagga gör det.

## Steg 6 – Lämna in på itslearning

Lämna in länken till ert repo (repo-URL:en, t.ex.
`https://github.com/<användarnamn>/<repo>`) på itslearning under
inlämningen **"Milstolpe 1"**.

## Efter labben – skaffa ert eget CSC-projekt (deadline: före session 7)

Molnet ni jobbar i från session 7 (M7–M9) är **CSC cPouta**, och där hör
allt — VM:ar, IP-adresser, kvoter — till ett **projekt**. Ni skaffar ett
eget: **ett projekt per par** (solo = ett eget), där **båda** är
medlemmar. Det är inte dagens labbjobb, men gör det **den här veckan**:
godkännandena går via CSC och kan ta några dagar, och den 29.9 ska det
bara fungera.

Den ena i paret (kalla hen "projektägaren") gör steg 1–3, den andra
gör steg 4. Allt sker i **MyCSC** (<https://my.csc.fi>, logga in med
Haka):

1. **Skapa projektet:** vänstermenyn **Projects** → **+ New project** →
   **Project category** = *Academic* → fyll i **Project name** (t.ex.
   `devops-<era-förnamn>`) och **Project description** (skriv att det är
   för en DevOps-kurs på Arcada) → godkänn användarvillkoren → **Create
   project**. Avgiftsfritt för finländsk högskoleutbildning.
2. **Aktivera cPouta på projektet:** öppna projektet → under **Services**
   → välj **Pouta** → läs och godkänn villkoren → **Apply for Access**.
   Ni får ett mejl när accessen är på plats. *Utan det här steget finns
   projektet, men <https://pouta.csc.fi> släpper inte in er.*
3. **Bjud in par-kompisen:** i projektet, under **Members** →
   **Manage invitation link** → kopiera länken och skicka den till
   den andra.
4. **Par-kompisen går med:** öppna länken, logga in i MyCSC med Haka och
   klicka **Apply project membership**. Säg till projektägaren att du
   gjort det — CSC skickar **inget** mejl om ansökningar.
5. **Projektägaren godkänner:** projektet → **Members** →
   **Membership applications** → godkänn.
6. **Par-kompisen godkänner tjänsten:** projektet → **Services** → raden
   med Pouta (markerad med en klocksymbol) → godkänn villkoren →
   **Accept**. Varje medlem godkänner för egen del.

**Klart när:** båda kan logga in på <https://pouta.csc.fi> med Haka och
se **samma** projekt, och båda ser varandra i projektets medlemslista i
MyCSC. Det är exakt det ni verifierar i M7 (steg 11).

Fastnar ni — projektansökan avslås, cPouta-accessen dröjer, inbjudan
funkar inte — säg till läraren **direkt**, det är ett administrativt
ärende och inte något ni ska gissa er igenom.

Menytexterna ovan är hämtade från CSC:s dokumentation:
<https://docs.csc.fi/accounts/how-to-create-new-project/>,
<https://docs.csc.fi/accounts/how-to-add-service-access-for-project/>,
<https://docs.csc.fi/accounts/how-to-add-members-to-project/>.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] Repot finns på GitHub och sidan visar *"generated from ..."* under
      namnet (= skapat från template, inte fork, inte tomt).
- [ ] `git log` visar din commit med **rätt författarnamn**.
- [ ] Committen syns på GitHub — pushen gick fram.
- [ ] Taggen syns på GitHub: repo-sidan → **Tags** (fliken bredvid
      *Branches*), eller `git tag` i terminalen (codespace eller lokalt).
- [ ] **I par:** båda är collaborators och båda har minst en egen commit.
- [ ] Läraren är inbjuden som collaborator på repot.
- [ ] GitHub-länken är inlämnad på itslearning under **"Milstolpe 1"**.

Dessutom, **före session 7** (inte idag): ert eget CSC-projekt enligt
avsnittet **"Efter labben"** ovan.

## Vanliga problem

**"Use this template"-knappen finns inte.**
Du tittar på fel repo (t.ex. ditt eget eller en fork). Gå till kursens
template-URL från steg 2.

**`git status` visar `.devcontainer/devcontainer-lock.json` som ändrad — innan du gjort något.**
Förväntat, inget att oroa sig för: det är en lockfil för devcontainer-features
(samma idé som `package-lock.json`) och Codespace-bygget kan lösa dem till
nyare versioner än lockfilen anger. Ta med den i commit 1 som vanligt —
gitignorea den **inte**, den ger reproducerbara miljöer.

**`git push` frågar efter lösenord och vägrar.**
GitHub tar inte emot kontolösenord över HTTPS. I Codespaces uppstår det
här aldrig (autentiseringen är inbyggd, `gh` finns redan i devcontainern)
— kör du lokalt: logga in med GitHub CLI (`gh auth login`, installera den
separat om du inte redan har den), eller sätt upp en personal access
token / SSH-nyckel.

**Committen visar fel namn, eller "unknown author".**
`git config user.name`/`user.email` var fel när du committade — fixa enligt
steg 4. Redan committat med fel namn? Gör om senaste committen med rätt
identitet: `git commit --amend --reset-author --no-edit` och pusha igen.

**GitHub visar committen som grå/utan länk till din profil.**
E-posten i `git config user.email` matchar ingen adress på ditt
GitHub-konto. Lägg till adressen på GitHub (*Settings* → *Emails*) eller
byt till din GitHub-adress och gör om committen.

**Taggen syns inte på GitHub.**
Du glömde pusha den: `git push origin m1-repo`. Kontrollera med `git tag`
att den finns i din arbetskopia (codespace eller lokalt) först.

**Taggen pekar på fel commit.**
Ni taggade för tidigt. Så gör man i branschen: en publicerad tagg flyttas
inte, man drar tillbaka den och sätter en ny.

```bash
git tag -d m1-repo
git push --delete origin m1-repo
git tag m1-repo
git push origin m1-repo
```

(`git tag -f` gör samma sak i ett kommando, men gömmer bort-och-om-steget
i en flagga — vi visar inte det som recept.)

**I par:** om par-kompisen redan hämtat den gamla taggen uppdateras den
inte hos hen med en vanlig `git fetch` — hen behöver `git fetch --tags
--force` (eller `git tag -d m1-repo` lokalt och sedan `git fetch`).
Läraren kontrollerar taggen på GitHub, så det är oftast ofarligt.

**Par-kompisen eller läraren ser inte repot / kan inte pusha.**
Inbjudan som collaborator måste **accepteras** — kolla mejl/aviseringar på
GitHub (gäller både par-kompisen och läraren). Före accept är repot inte
synligt/skrivskyddat för dem.
