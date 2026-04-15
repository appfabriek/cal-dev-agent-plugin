---
name: cal-review
description: Review C/AL code for quality, patterns, and common pitfalls
allowed-tools: Bash, Read, Write, Glob
---

# CAL Review

Review C/AL code op kwaliteit, patronen en veelgemaakte fouten.

## Input

$ARGUMENTS — wat te reviewen:
- (leeg) — review alle gewijzigde objecten (git diff)
- "COD50000" of een bestandsnaam — één specifiek object
- "objects/" — alle objecten in de map

## Instructies

### Stap 0 — Laad kennis

```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-guidelines.md" 2>/dev/null | head -1
```

### Stap 1 — Lees de te reviewen code

```bash
# Gewijzigde objecten
git diff HEAD objects/ -- "*.txt"

# Specifiek bestand
cat "objects/COD50000 - My Codeunit.txt"
```

### Stap 2 — Review checklist

Controleer op elk van deze categorieën:

#### Correctheid
- [ ] FlowFields worden aangeroepen met `CALCFIELDS` vóór gebruik
- [ ] `FINDSET` altijd gevolgd door `REPEAT ... UNTIL NEXT = 0`
- [ ] `LOCKTABLE` vóór `GET` als record gewijzigd wordt
- [ ] Tekstvelden afgekapt met `COPYSTR(..., 1, MAXSTRLEN(...))`
- [ ] Guard flags voor recursieve `VALIDATE` aanroepen — en altijd gereset
- [ ] `EXIT` in alle code-paden van procedures met return value

#### Foutafhandeling
- [ ] `ERROR('')` (stille annulering) is bewust — niet per ongeluk
- [ ] Gebruikersmeldingen zijn duidelijk en actiegerich
- [ ] Geen swallowed errors (lege `IF ... THEN ;` na error-gevoelige operaties)

#### Performance
- [ ] Geen `FINDSET` in een loop die al in een `FINDSET` zit (geneste loops)
- [ ] `CALCFIELDS` niet in een loop tenzij noodzakelijk
- [ ] `SETRANGE`/`SETFILTER` zo specifiek mogelijk vóór `FINDSET`
- [ ] Geen onnodige `COMMIT` in middelste logica

#### Naamgeving & structuur
- [ ] Variabelenprefixen consistent: `l` lokaal, `g` globaal, `p` parameter
- [ ] Procedures hebben een duidelijke naam (geen `DoStuff`, `Process`)
- [ ] Version List bijgewerkt met de juiste tag

#### BC14 / migratiegereedheid
- [ ] Directe tabelaanpassingen zijn gemarkeerd — overweeg event-subscriber patroon
- [ ] Geen hardcoded company-namen
- [ ] Geen gebruik van verouderde objecttypen (Form, Dataport) tenzij onvermijdelijk

### Stap 3 — Rapporteer

Geef feedback per categorie. Onderscheid:
- **Blocker**: moet opgelost worden vóór import (compilatiefout, data-corrupt risico)
- **Waarschuwing**: potentieel probleem, bespreek met developer
- **Suggestie**: verbetering maar geen vereiste

Voorbeeld output:
```
## Review: COD50000 - Sales Helper

**Blockers (1):**
- Regel 45: CALCFIELDS ontbreekt vóór gebruik van "Balance (LCY)" — geeft altijd 0

**Waarschuwingen (2):**
- Regel 78: ERROR('') stille annulering — bewust?
- Regel 92: FINDSET in loop zonder NEXT controle

**Suggesties (1):**
- Version List niet bijgewerkt (nog MYAPP1.04, wijziging is voor 1.05)
```

## Regels

- Wees specifiek: regelreferentie + uitleg waarom het een probleem is
- Prioriteer blockers — die moeten opgelost worden vóór deployment
- Geen perfectie nastreven: focus op correctheid en duidelijke bugs
