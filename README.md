# Kursmaterial — DevOps: från commit till produktion

Det här repot innehåller kursmaterialet: föreläsningsslides, lab-instruktioner,
onboarding och betygsrubriken. Det klonas automatiskt in i din devcontainer som
`course-material/`, så materialet finns bredvid din kod när du jobbar.

## Innehåll

```
slides/       Föreläsningsslides som Marp-markdown + renderad PDF
labs/         Lab-instruktionen för varje milstolpe (M1–M9)
onboarding/   Kom-igång-checklista: konton, template-repo, verktyg
grading/      Betygsrubriken: betyg 1–5, milstolparna och de tre spåren
```

## Materialet fylls på under kursen

Materialet publiceras **inför varje föreläsning**, inte allt på en gång. Nya
filer tillkommer alltså löpande — kör `git pull` i `course-material/` före varje
föreläsning och lab så har du den senaste versionen.

Börja med `onboarding/student-onboarding.md` och därefter lab-instruktionen för
milstolpen ni jobbar med.

## Läsa slides

Varje deck finns både som markdown (`slides/<deck>.md`) och som färdig PDF i
samma mapp. Vill du rendera själv, t.ex. efter en `git pull`:

```bash
npx @marp-team/marp-cli slides/<deck>.md --pdf --allow-local-files
```

`slides/referens-git-cheatsheet.md` hör inte till någon enskild föreläsning —
det är en uppslagsdeck (commit-flöde, taggning, felrecept) att ha uppe under
labbarna.
