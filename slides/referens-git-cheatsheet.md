---
marp: true
theme: default
paginate: true
title: "Referens — Git cheat sheet"
footer: "Från commit till produktion · Referens · Git cheat sheet"
---

# Git cheat sheet

**Referens — ha uppe under labbarna, inte en föreläsning.**

## Arbetsordningen (1/2) — branch till PR

1. `git switch -c <branch>` — ny branch (inte `checkout -b`)
2. Gör ändringen
3. `git add <fil>`
4. `git commit -m "..."`
5. `git push -u origin <branch>` — första pushen av branchen
6. Öppna PR på GitHub

---

## Arbetsordningen (2/2) — review till tagg

7. Buddy-review + approve
8. Merge på GitHub — radera branchen
9. `git switch main && git pull`
10. `git tag mX-...` — **på main, efter merge**, när milstolpen är klar
11. `git push origin mX-...` — separat push

---

## Commit-flödet

```bash
git status                          # se vad som ändrats
git add <fil>
git commit -m "Beskriv varför, inte bara vad"
git push                            # push -u origin <branch> vid första pushen
```

Bra commit-meddelande: **varför**, inte bara vad du ändrade.

---

## Taggning

```bash
git tag mX-...                      # på rätt commit, oftast main efter merge
git push origin mX-...              # taggar pushas SEPARAT — git push tar inte med dem
git tag -l                          # lista lokala taggar
```

Kontrollera på GitHub: repo → **Tags**.

---

## Tagg satt för tidigt? Flytta den

```bash
git tag -d mX-...                   # radera lokalt
git push origin --delete mX-...     # radera på GitHub
git tag mX-...                      # på rätt commit
git push origin mX-...              # pusha igen
```

Aldrig `git tag -f` — delete + återskapa, inga force-flaggor.

---

## "src refspec ... matches more than one"

Branch och tagg heter samma sak — Git vet inte vilken du menar.

```bash
git push origin tag mX-...                   # otvetydigt: pusha TAGGEN, inte branchen
git push origin --delete refs/heads/mX-...   # otvetydigt: radera BRANCHEN, inte taggen
```

---

## "fatal: tag already exists"

```bash
git tag -d mX-...                   # radera den lokala taggen med samma namn
git tag mX-...                      # skapa på rätt commit
git push origin mX-...              # pusha
```
