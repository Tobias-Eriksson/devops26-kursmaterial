# Betygsrubrik — Från commit till produktion

**Delas ut dag 1. Uppdaterad 13.10.2026:** spårens final task är borttagen ur kursen — kursen består av milstolparna M1–M9. Betyg 1–5. Kort version: **poängen avgör betyget — M1–M9 är hela grunden, bonusen ger höjden.**

## Poängskala (max 109 p)

| Del | Poäng |
|---|---|
| Milstolpar M1–M9 | 0–11 p per milstolpe (helhetsbedömning) = max **99** |
| Bonus: repo-hygien, commit-historik, PR-disciplin, fungerande kedja | **+10** |
| **Max** | **109** |

**Betygsgränser:** 0–50 Underkänt · 51–60 = 1 · 61–70 = 2 · 71–80 = 3 · 81–90 = 4 · 91+ = 5

Tumregel: varje fullt genomförd milstolpe är värd ungefär ett betygssteg. Alla nio milstolpar perfekt = 99 p = betyg 5; till och med M8 perfekt = 88 p = betyg 4; till och med M6 perfekt = 66 p = betyg 2.

**Hårda krav utöver poängen:**

- Godkänt kräver **minst 51 p OCH individuell reflektion** (~1 sida) om produktens
  livscykel + "vad går sönder utan automation?".

## Deadlines — och när bedömningen sker

| När | Vad |
|---|---|
| **Ons 28.10 23:59** (hård slutdeadline) | Allt ska vara klart och committat: M1–M9 inklusive milstolpsbevis (taggar och `inlamning/mN-*.md`) **och** den individuella reflektionen (`inlamning/REFLEKTION.md`; länken lämnas också in på itslearning). |

**Vad bedöms var:**

- **Allt bedöms ur repot.** Milstolpepoängen (0–11 p per milstolpe) samt
  bonusen läses ur repot efter deadline, alltså från och med **to 29.10**.
  Committa allt ni vill ha bedömt före ons 28.10 23:59 — det som inte
  finns i repot då räknas inte.
- Har ni alla nio milstolpar klara och reflektionen inlämnad är ni klara
  med kursen — det finns ingen slutuppgift utöver milstolparna.

## Milstolpar M1–M9

| # | Milstolpe | Bevis i repot |
|---|---|---|
| M1 | Konton, repo från template, första commit | Repo + första push |
| M2 | Arbetsflöde med branch protection, review | Mergad PR med review (solo: dokumenterad självgranskning i PR:en) |
| M3 | Dockerfiles frontend+backend, compose lokalt, push GHCR | Båda images i GHCR |
| M4 | Testsvit, utdelad bugg hittas via test | Grön testsvit |
| M5 | Actions kör lint+test på varje PR | Grön körning i Actions |
| M6 | CI bygger+pushar images automatiskt | Commit på main → nya images |
| M7 | VM på cPouta, appen svarar på floating-IP | Appen svarar på floating-IP/nip.io |
| M8 | Terraform återskapar M7 från noll | `terraform apply` → fungerande miljö |
| M9 | CD: Actions deployar via SSH | Merge till main → produktion uppdateras |

## Bonus +10 p — vad "kvalitet" betyder konkret

- **Repo-hygien** — rimlig `.gitignore`, inga secrets committade, tydlig struktur.
- **Commit-historik** — många små commits med tydliga meddelanden, inte en jättecommit i slutet.
- **PR-disciplin** — featurebranches, PR med review innan merge till main.
- **Fungerande kedja** — varje milstolpe har verifierbart bevis (grön pipeline, svarande URL,
  `terraform apply` som funkar) — inte bara "det funkade på min dator".

## Individuell bedömning i par

Ni får jobba i par och dela repo — men **betyget är alltid individuellt**. Läraren
bedömer varje person via:

- **Git-historiken** — vem har committat vad, hur ofta, vilken kvalitet.
- **Milstolpsbevisen** — i `inlamning/mN-*.md` framgår vem i paret som gjort
  vad, så att bedömningen ser vem som förstått vad.
