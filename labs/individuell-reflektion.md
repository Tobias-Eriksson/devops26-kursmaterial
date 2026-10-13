# Individuell reflektion (~1 sida)

Krav för godkänt, oavsett betyg 1–5 — se `grading/grading-rubric.md` och
`kursplan.md` (raden *"Praktisk insikt i servicelivscykeln
(förhållningssätt)"* i tabellen "Koppling till läranderesultaten"). Det här
är den delen av kursen som bedömer ert **förhållningssätt**, inte er kod:
har ni faktiskt förstått varför kedjan ser ut som den gör, inte bara hur
man kör kommandona. Det är också kursens avslutande bearbetning — någon
gemensam retro hålls inte, ni gör den var för sig i den här texten.

## Vem skriver

**Alltid individuellt**, även om ni jobbat i par och delat repo. Två
personer i samma par kan ha helt olika reflektioner — ni har troligen inte
gjort exakt samma saker eller dragit samma lärdomar, och det är poängen.
Kopiera inte varandras text.

## Vad ni ska skriva om

Två delar, ungefär lika långa:

**1. Produktens livscykel.** Beskriv resan er tjänst gjort genom kursen —
från första commit (M1) till att den svarar i produktion via en
automatiserad kedja (M9). Vad hände i vilket steg, vilka
verktyg löste vilket problem, och vad förändrades i hur ni själva jobbade
när ni gick från att göra saker för hand (session 1–2 och de manuella
stegen i M7) till att kedjan gjorde det åt er (M5–M9)?

**2. Vad går sönder utan automation?** Välj **minst en konkret punkt** i
er egen kedja — t.ex. testsviten, CI-pipelinen, IaC, eller CD-steget —
och beskriv specifikt vad som skulle gå fel, ta längre tid, eller bli
riskablare om just den biten gjordes för hand igen. Använd egna
erfarenheter från kursen som bevis, inte bara ett allmänt påstående om att
"automation är bra".

## Format

- **~1 sida** (ungefär 400–600 ord). Fritext, inget mallformulär.
- Svenska.
- Committa filen som `inlamning/REFLEKTION.md` i ert eget repo — samma
  mapp som milstolpsbevisen, samma repo som resten av kedjan, så
  historiken över era commits finns redan där som stöd åt läraren.

## Deadline

**Senast onsdag 28.10 kl. 23:59** — samma hårda slutdeadline som
milstolparna (repot bedöms från to 29.10, se `grading/grading-rubric.md`).
Skriv den när M9 är i hamn: då har ni hela resan, från första commit till
fungerande kedja, att reflektera över.

Committa och pusha filen i ert eget repo före deadlinen. Den är lätt att
glömma bort när man känner sig färdig, så lägg in den i kalendern redan nu.
En reflektion som inte finns i repot 28.10 23:59 räknas inte som inlämnad,
och utan den blir det inget godkänt oavsett poäng.

Reflektionen finns också som inlämning på itslearning: lämna in länken till
filen i ert repo där. Det är repot som bedöms.
