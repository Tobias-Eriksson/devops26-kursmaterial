# Betygsrubrik — Från commit till produktion

**Delas ut dag 1.** Betyg 1–5. Kort version: **poängen avgör betyget — M1–M9 är grunden, final task ger höjden.**

## Poängskala (max 110 p)

| Del | Poäng |
|---|---|
| Milstolpar M1–M9 | 0–9 p per milstolpe (helhetsbedömning) = max **81** |
| Final task (valt spår A/B/C) | 0–19 p — fungerande demo, robusthet, dokumentation, demofrågor |
| **Kärnsumma** | **100** |
| Bonus: repo-hygien, commit-historik, PR-disciplin, fungerande kedja | **+10** |
| **Max** | **110** |

**Betygsgränser:** 0–50 Underkänt · 51–60 = 1 · 61–70 = 2 · 71–80 = 3 · 81–90 = 4 · 91+ = 5

**Hårda krav utöver poängen:**

- Godkänt kräver **minst 51 p OCH individuell reflektion** (~1 sida) om produktens
  livscykel + "vad går sönder utan automation?".
- Betyg **5** kräver final task med **fungerande live-demo**. Demon hålls som
  **individuellt bokade 30-minuterspass** i slutet av kursen.

## Deadlines — och när bedömningen sker

| När | Vad |
|---|---|
| **Session 10, fr 9.10** (rekommenderat) | M1–M9 bör vara klara. Inte ett hårt krav — men spårvalet blir ett informerat val först när grunden finns, och projektveckan har ingen undervisning. |
| **Ert bokade demopass, 19–20.10** (hård deadline) | Allt ska fungera: M1–M9 **och** spårets final. Demon körs på det som finns då. |
| **Ti 27.10 23:59** (touch-up-fönster) | Efter demot får repot putsas — dokumentation, milstolpsbevis i `inlamning/`, småfixar. |
| **Fr 23.10** | Individuell reflektion inlämnad (egen artefakt, egen deadline — påverkas inte av touch-up-fönstret). |

**Vad bedöms var:**

- **Demopasset** bedömer det som går att visa live: kedjan från commit till
  produktion, spårets final och era svar på demofrågorna — på repots skick
  vid passet. Det som inte fungerar då går inte att demonstrera i efterhand.
- **Milstolpe- och final task-poängen** (0–9 p per milstolpe, 0–19 p för
  finalen) samt bonusen läses ur **repot efter touch-up-fönstret**, alltså
  från och med **ons 28.10**. Committa allt ni vill ha bedömt före
  ti 27.10 23:59.

## Milstolpar M1–M9

| # | Milstolpe | Bevis i repot |
|---|---|---|
| M1 | Konton, repo från template, första commit | Repo + första push |
| M2 | Arbetsflöde med branch protection, review | Mergad PR med review |
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

## Final task — spåren (0–19 p)

- **A — Drift & observability:** Prometheus/Grafana, lasttest, felsök ett injicerat fel + postmortem.
  - Postmortemet har en egen PromQL-query (eller ännu hellre en
    dashboard-panel) relevant för just deras incident: queryn/panelen
    finns, skärmdumpen finns, och tolkningen stämmer med vad som
    faktiskt hände. Sektion 7:s exempelqueries räknas inte som svar.
  - Egen larmregel i `alert.rules.yml`, självtestad: villkoret är
    framkallat och skärmdumpen visar `Firing`. Regeln kan vara skriven
    före eller efter incidenten — kommer den efter, förklarar
    postmortemet vad den skulle ha visat.
  - Postmortemets "Vad missade övervakningen?"-sektion analyserar
    korrekt vad övervakningen såg/missade, inklusive den egna regelns
    roll under eller efter incidenten.
- **B — GitOps & Kubernetes:** Rahti + Flux, ändring i produktion enbart via mergad PR mot manifest-repot.
- **C — DevSecOps & kvalitet:** två miljöer, CI-gates (Trivy/CodeQL/Dependabot), stoppa en sårbar
  ändring före prod.

## Individuell bedömning i par

Ni får jobba i par och dela repo — men **betyget är alltid individuellt**. Läraren
bedömer varje person via:

- **Git-historiken** — vem har committat vad, hur ofta, vilken kvalitet.
- **Demo-frågor** — i det bokade demopasset får båda i paret frågor, var för sig,
  och måste själva kunna redogöra för sin del av kedjan.
