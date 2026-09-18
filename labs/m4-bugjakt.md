# M4 – Bugjakt: bevisa buggen med ett test

Den här övningen hör till **session 4** (Automatisk testning, M4). Ni jobbar
i par, ca 30–45 minuter, och avslutar med en gemensam genomgång.

**Uppdatera kursmaterialet:** `cd course-material && git pull`.

## Läget

En kollega har shippat en ny feature till er app: en liten summeringsrad
under listan som visar hur många anteckningar som finns och hur många
tecken de innehåller totalt. Pull requesten är granskad och mergad. Den
kom med test. Testsviten är grön.

Nu har en **användare rapporterat ett fel:**

> *"Teckenräknaren visar fel värde efter att jag tagit bort en anteckning.
> Antalet anteckningar stämmer, men tecknen gör det inte."*

Läraren visar felrapporten live på föreläsningen — samma klick, samma
siffror. Er uppgift är inte att gissa vad som är fel, utan att **bevisa**
det: skriv ett test som gör exakt det användaren gjorde och som failar på
den nuvarande koden. Hitta sedan orsaken i koden, fixa den, och se samma
test bli grönt.

## Regler

1. Buggen räknas som bevisad först när ni har ett test som **failar** på
   den nuvarande koden och som **passerar** när buggen är fixad. Ett
   påstående i chatten räknas inte — inte ens ett korrekt påstående.
2. Fixa inte först och skriv test efteråt. Skriv testet först, se det bli
   rött, fixa sedan. (Det är hela poängen: testet ska bevisa att felet
   fanns, och att just er fix tog bort det.)
3. Ni får läsa all kod ni vill.

## Kom igång

Kollegans feature ligger som **en enda commit** på branchen `bughunt-m4`
i kursens template-repo — inte i ert eget. Ni hämtar den commiten och
lägger den ovanpå **er** `main`, på en egen branch. I er codespace (eller
lokala klon):

```bash
cd <ditt-repo>

git switch main && git pull
git switch -c bughunt-m4

# Kursens template-repo som extra remote (samma URL som i M1)
git remote add upstream https://github.com/Tobias-Eriksson/devops26-template
git fetch upstream

# Lyft över kollegans commit till er branch
git cherry-pick upstream/bughunt-m4

# Branchen ska ligga i ERT repo — ni har ingen skrivrättighet i template-repot
git push -u origin bughunt-m4
```

**Varför `cherry-pick` och inte `merge`?** Ert repo skapades med *Use this
template* i M1. Det kopierade filerna, men **inte historiken** — ert repo
och template-repot har inte en enda commit gemensamt. Försöker ni merga
vägrar Git (`refusing to merge unrelated histories`), och checkar ni ut
template-branchen rakt av byter ni ut era egna M1–M3-ändringar mot
templatens filer. `git cherry-pick` gör det ni faktiskt vill: tar
**ändringen** i kollegans commit och lägger den som en ny commit ovanpå
er egen historik. Titta med `git log --oneline -3` — kollegans commit
ligger nu överst, ovanpå era egna.

Innan ni börjar jaga något ska sviten vara grön och appen igång:

```bash
# Testsviten ska vara grön innan ni börjar (antalet test beror på vad ni
# själva skrev i M2 — det viktiga är att inget är rött)
docker compose run --rm --build backend pytest -q

# Starta appen och klicka runt i den
docker compose up -d --build
```

Öppna <http://localhost:8080> — appen ska svara. (I Codespace: se M3,
Steg 0 för hur ni når den vidarebefordrade porten via Ports-fliken;
lokalt: `open http://localhost:8080` på macOS eller besök adressen
direkt.)

**Reproducera felrapporten för hand först**, så att ni vet vad ni ska
bevisa: lägg till anteckningen `milk`, lägg till `bread`, läs
summeringsraden. Ta bort `bread`. Läs raden igen. Vilket tal stämmer, och
vilket gör det inte? Räkna själva.

> 📸 **Kom ihåg skärmdump till inlämningen:** appen efter borttagningen — listan med en anteckning kvar och summeringsraden under den.

API:et nås via `http://localhost:8080/api/...` (nginx proxar vidare till
backend). Hela API-ytan syns i `backend/app/main.py` — den filen är kort,
läs den. Befintliga test ligger i `backend/tests/test_main.py`.

## Testfilen finns redan

Filen `backend/tests/test_bugjakt.py` **finns redan** i `backend/tests/` —
den kom med cherry-picken. Importerna och `client` är på plats, och
byggstenarna ni behöver står som kommentarer i filen: skapa en anteckning,
få tillbaka objektet, ta bort en anteckning, hämta summeringen. Kopiera
och kombinera. Filen innehåller inget test ännu, så `pytest` säger
`no tests ran` tills ni skrivit ert första.

Kör bara er egen fil medan ni jobbar:

```bash
docker compose run --rm --build backend pytest tests/test_bugjakt.py -q
```

## Pytest-recept

Allt ni behöver finns att kopiera i `backend/tests/test_main.py`:

- **Varje test är en funktion vars namn börjar med `test_`.** Namnet är
  hela registreringen: heter funktionen `test_...` hittar pytest den. Ge
  den ett namn som säger vad den kontrollerar.
- **`assert` är kontrollen.** `assert response.json() == {...}` betyder
  "det här måste vara sant, annars failar testet". Ett test utan `assert`
  bevisar ingenting.
- **`client.post(...).json()` ger tillbaka det skapade objektet**, och
  `["id"]` på det är vad ni tar bort med — precis som `test_delete_item`
  gör i `test_main.py`.
- **Varje test börjar tomt.** `backend/tests/conftest.py` nollställer
  appen automatiskt före varje test, så ni behöver inte städa efter er
  och inget annat test påverkar ert.
- **Räkna ut det förväntade värdet för hand innan ni kör.** Ni vet vad
  `milk` och `bread` innehåller. Vad *borde* summeringen säga efter att
  `bread` är borta?
- **När ett test failar visar pytest båda sidorna:** vad koden faktiskt
  svarade och vad ni förväntade er. Läs båda. Bestäm er för **vilken sida
  som har fel** innan ni ändrar något — ibland är det testet som räknat
  fel, och det är också ett resultat.

## Så jobbar ni

1. **Reproducera i test.** Gör i testet exakt det användaren gjorde: skapa
   två anteckningar, ta bort den ena, hämta summeringen, `assert` det ni
   räknat ut för hand. Kör er fil. Rött? Då har ni beviset — **spara
   utskriften** (kopiera terminalen till `inlamning/m4-tests.md`, se
   nedan). Grönt? Då gjorde testet inte samma sak som användaren — läs
   felrapporten igen.
2. **Följ spåret från skärmen in i koden.** Summeringsraden ritas i
   `frontend/app.js` — vilken adress hämtar den sina tal från? Hitta
   funktionen i `backend/app/main.py` som svarar på den adressen. Ställ
   sedan samma fråga om **vart och ett** av de två talen: var räknas det
   ut, och vilka operationer i filen påverkar det? Ett av talen stämmer
   alltid. Varför gör det andra inte det?
3. **Fixa.** Gör den minsta ändring som får ert test att bli grönt. Kör
   sedan hela sviten — inget annat får ha gått sönder.

## Testdesign – idéer när ni kör fast

Fundera på vilka **frågor** ni ställer till koden, inte på hur många rader
test ni skriver:

- **Sekvenser:** en enskild operation kan vara rätt medan två i följd blir
  fel. Det är precis vad användaren råkade på — och precis vad kollegans
  test aldrig prövade.
- **Invarianter:** vad måste vara sant om systemet *hela tiden*, oavsett
  vilka anrop som gjorts? Summeringen ska alltid stämma med listan. Ett
  test som kontrollerar *det*, i stället för ett handräknat värde, fångar
  varje framtida bugg av samma sort.
- **Randfall:** tomt tillstånd, ett enda objekt, ett id som inte finns.

## Inlämningsrapport

Förklara följande i `inlamning/m4-tests.md`:

1. Testet som failar (klistra in output från `pytest`).
2. En förklaring: **varför missade code review, den gröna testsviten och
   den mergade pull requesten den här buggen?** Tre skyddsnät släppte
   igenom den — vad tittade vart och ett av dem på, och vad tittade de inte
   på?
3. Er fix, och samma test grönt.
4. Vad hade behövts för att den här buggen aldrig skulle ha nått main?

**AI-verktyg:** samma regel som i M2 — Claude Code, Codex och Gemini i
devcontainern får hjälpa er med pytest-**syntaxen**. Men punkt 2 och 4
svarar ni själva på, med egna ord — det är resonemanget som bedöms, inte
koden.

## Leverera via PR — M2-flödet gäller

Buggfixen är en kodändring som alla andra: den går in i `main` via en pull
request, inte direkt. På branchen `bughunt-m4`:

```bash
git add backend/app/main.py backend/tests/test_bugjakt.py inlamning/m4-tests.md
git commit -m "Fix summary character count after deleting a note"   # varför, inte bara vad
git push
```

Öppna PR:en mot **er egen** `main` (GitHub visar bannern *Compare & pull
request*). Skriv i beskrivningen vad buggen var och lägg till **"Så testar
du"**: kommandot som kör ert test. **Par:** buddyn granskar enligt M2 steg
4 — checkar ut branchen, kör sviten, kontrollerar att testet faktiskt
täcker sekvensen ur felrapporten. **Solo:** dokumenterad självgranskning
som i M2 (beskrivning + minst en egen radkommentar). Merga, radera
branchen.

**Tagga sedan på `main`**, när fixen är mergad:

```bash
git switch main && git pull
git tag m4-tests && git push origin m4-tests
```

**Bevisa det osynliga:** den röda testkörningen innan fixen och
resonemanget om varför buggen inte syntes (punkt 1 och 2 i
inlämningsrapporten) syns inte i repot om ni bara säger dem högt.
Skriv ner dem i `inlamning/m4-tests.md` — klistra in `pytest`-utskriften
från den röda körningen, några meningar om skyddsnäten som missade, och
skärmdumpen från appen — och committa filen tillsammans med fixen.
Formatet visas i `inlamning/m0-exempel.md`.

## Verifiera

Klart betyder att allt det här stämmer:

- [ ] `docker compose run --rm --build backend pytest -q` är **grön** på
      `main`
- [ ] Ett test i `backend/tests/test_bugjakt.py` som **failade** på den
      ofixade koden finns i repot, och den röda körningen är sparad i
      `inlamning/m4-tests.md` som bevis
- [ ] Fixen är applicerad, samma test är nu **grönt**
- [ ] En mergad PR från `bughunt-m4` till `main` finns under
      **Pull requests** → *Closed* — granskad av buddyn, eller med
      dokumenterad självgranskning (solo)
- [ ] Taggen `m4-tests` pekar på `main` efter mergen och syns under
      **Tags** (eller `git tag` lokalt)
- [ ] Det osynliga arbetet är dokumenterat i `inlamning/m4-tests.md`

## Vanliga problem

**`CONFLICT (content): Merge conflict in backend/app/main.py` vid
cherry-picken.** Det händer om ni i M2 lade er `GET /api/items/{item_id}`
*ovanför* funktionen som skapar anteckningar — kollegans commit lägger sin
nya route på samma ställe. Öppna filen: mellan `<<<<<<< HEAD` och
`=======` står **er** route, mellan `=======` och `>>>>>>>` står
**kollegans** (`/api/items/stats`). Behåll **båda** — men ordningen
spelar roll: routen `/api/items/stats` måste stå **ovanför** routerna med
`{item_id}`. En route med fast text ska alltid stå före en route med
parameter i samma position, annars tolkar FastAPI ordet `stats` som ett
`item_id`. Ta bort markörraderna, spara, och avsluta:

```bash
git add backend/app/main.py
git cherry-pick --continue
```

Symptom på fel ordning: `http://localhost:8080/api/items/stats` svarar
**422** (`Input should be a valid integer`) och kollegans test
`test_stats_counts_items` är rött redan innan ni börjat. Det är en
merge-artefakt, **inte** den rapporterade buggen — flytta routen, kör
sviten igen.

**`fatal: refusing to merge unrelated histories`.** Ni körde `git merge`
eller `git pull` mot `upstream` i stället för `git cherry-pick`. Ert repo
delar ingen historik med template-repot (se *Kom igång*). Avbryt med
`git merge --abort` om Git står mitt i en merge, och kör cherry-pick-raden.

**`error: remote upstream already exists`.** Ni har redan lagt till
remoten (t.ex. om ni gjort om steget). Hoppa över raden och fortsätt med
`git fetch upstream`.

**`fatal: a branch named 'bughunt-m4' already exists`.** Antingen har ni
kört *Kom igång* en gång redan, eller så slog ni på *Include all
branches* när ni skapade repot i M1 — då följde template-repots branch
med, **utan** ert M1–M3-arbete. Ta bort den (`git branch -D bughunt-m4`,
och `git push origin --delete bughunt-m4` om den även ligger på GitHub)
och gör om *Kom igång* från `git switch -c bughunt-m4`.

**`no tests ran` fast jag skrivit ett test.** Funktionsnamnet måste börja
med `test_`, filen måste vara sparad — och `--build` måste vara med i
kommandot: utan volymmontering i compose-filen ser containern inte er
ändring förrän imagen byggts om.

**Mitt test blev grönt direkt.** Då gör det inte det användaren gjorde.
Kollegans eget test är också grönt — det skapar bara anteckningar. Vilket
steg i felrapporten saknas i ert test?

**Jag fixade koden innan testet var skrivet.** Då är det inte bevisat att
testet fångar felet. Lägg undan fixen tillfälligt (`git stash`), kör
testet och se det rött, ta tillbaka fixen (`git stash pop`) och se det
grönt. Den röda körningen är den ni sparar.

**Efter omstart av appen visar summeringen rätt igen.** Appen håller allt
i minnet, så en omstart nollställer både listan och summeringen — felet
försvinner tills nästa borttagning. Buggen är inte slumpmässig: den är
reproducerbar från ett känt starttillstånd, och det är exakt vad ert test
ger er som en manuell klicksession inte gör.

**Jag taggade `m4-tests` på branchen, inte på `main`.** Taggen ska peka
på `main` efter mergen. Flytta den — inga force-flaggor, radera och
återskapa (receptet i M2:s *Vanliga problem* och
`slides/referens-git-cheatsheet.md`).

## För er som blir klara i tid

- **Ett test till.** Summeringen på en tom app (vad *borde* den säga?),
  eller: att ta bort ett id som inte finns ska ge 404 **och** lämna
  summeringen orörd. Rött först — eller grönt direkt? Båda är resultat.
- **Den bättre fixen.** Er första fix lagade troligen räknaren där den
  gick fel. Men varför finns det ett sparat tal att laga överhuvudtaget?
  Summeringen kan räknas ut från listan varje gång den efterfrågas —
  `sum(len(item.text) for item in _items)` — och då finns det inget
  andra tal som kan glida isär från listan, oavsett vilka operationer
  någon lägger till i framtiden. Läs kommentaren i koden om att
  summeringen ska vara "cheap": det är en optimering ingen mätt behovet
  av, i en app med tio anteckningar — och den är hela orsaken till att
  buggen kunde uppstå. Ta med det till genomgångens fråga: *vad hade
  behövts för att buggen aldrig nått main?*
