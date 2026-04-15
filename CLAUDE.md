# CLAUDE.md — cal-dev-agent-plugin

This file is loaded automatically when this plugin is installed. It provides guidance for C/AL development in BC14, NAV 2009–2018, and classic Navision environments.

## Wat is C/AL?

C/AL (Client/server Application Language) is de programmeertaal van klassiek Microsoft Dynamics NAV en Business Central 14 (de laatste versie met C/AL ondersteuning naast AL). C/AL wijkt fundamenteel af van het moderne AL:

| Aspect | C/AL | AL (modern) |
|--------|------|-------------|
| Broncode locatie | In de database als BLOB | `.al` bestanden op disk |
| Tooling | C/SIDE client of finsql.exe | VS Code + alc.exe |
| Compileren | Bij importeren in DB | alc.exe → `.app` bestand |
| Objectmodel | Direct aanpassen van base objects | Extensions, geen base-aanpassingen |
| Git | Vereist export naar `.txt` | Native (bestanden) |
| Deployment | Import-NAVApplicationObject | Publish-NAVApp |
| Versiebeheer | Object ID (geen affixes) | App ID + affixes |

## Ondersteunde platforms

| Platform | Versie | C/AL tooling |
|----------|--------|--------------|
| Business Central 14 | 14.x (on-prem) | C/SIDE + NAV Dev Shell + finsql.exe |
| NAV 2018 | 11.x | C/SIDE + finsql.exe |
| NAV 2017 | 10.x | C/SIDE + finsql.exe |
| NAV 2016 | 9.x | C/SIDE + finsql.exe |
| NAV 2015 | 8.x | C/SIDE + finsql.exe |
| NAV 2013 R2 | 7.1 | C/SIDE + finsql.exe |
| NAV 2013 | 7.0 | C/SIDE + finsql.exe |
| NAV 2009 | 6.x | Classic Client + finsql.exe |
| Navision 5.x | 5.x | Classic Client |
| Navision Attain / 4.x | 4.x | Classic Client |

## Remote Server Access

Gebruik altijd **NavAdminTool en finsql.exe via de self-hosted runner** (`bc-runner.yaml` van de bc-dev-agent-plugin). Nooit REST.

## Proactief kennis laden

Bij C/AL werkzaamheden, laad altijd eerst:
- `cal-guidelines.md` — C/AL syntax, triggers, datatypes, patronen
- `cal-objects.md` — objecttypen, ID-beheer, structuur

Op basis van de taak, laad ook:

| Taak | Knowledge file |
|------|---------------|
| Exporteren uit database | `cal-tools.md` |
| Importeren / compileren | `cal-tools.md` |
| Git-beheer, diffs, merges | `cal-git-patterns.md` |
| Migratie C/AL → AL | `cal-to-al-migration.md` |
| Navision / zeer oude versies | `cal-classic-navision.md` |

Knowledge files vinden:
```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ./.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "<filename>" 2>/dev/null | head -1
```

## Proactief antwoorden op veelgestelde vragen

Wanneer de gebruiker dit vraagt — lees `cal-tools.md` en handel direct:

| Vraag | Actie |
|-------|-------|
| "nieuw project opstarten" of "database initialiseren" | `/cal-init` — instance aanmaken, structuur, export |
| "haal laatste code op" of "sync met database" | `/cal-pull` — Modified=Yes objecten exporteren |
| "exporteer object X" of "haal codeunit Y op" | `Export-NAVApplicationObject` via NavAdminTool |
| "wat zit er in de database?" | `Get-NAVApplicationObjectProperty` + export |
| "importeer deze wijziging" | `Import-NAVApplicationObject` via NavAdminTool |
| "wat zijn de verschillen met productie?" | Export beide → git diff |
| "compileer object X" | Import met SynchronizeSchemaChanges |
| "welke versie van object X draait?" | `Get-NAVApplicationObjectProperty` |

## Development Workflow

1. **Export eerst** — haal het object op uit de database vóór je wijzigingen maakt
2. **Bewerk in tekst** — `.txt` formaat is diffbaar en trackbaar in git
3. **Valideer syntax** — C/AL heeft strikte whitespace-regels in het exportformaat
4. **Importeer en test** — compilatie vindt plaats bij import
5. **Git commit** — één object per commit waar mogelijk
6. **Deploy naar omgevingen** — in de juiste volgorde (afhankelijkheden eerst)

## Objectnaamgeving in git

Gebruik de conventie: `<type><id> - <naam>.txt`

Voorbeelden:
- `TAB50000 - Customer Extension.txt`
- `COD50000 - My Codeunit.txt`
- `PAG50000 - Customer Card Extension.txt`
- `REP50000 - My Report.txt`

Object type afkortingen: `TAB` (Table), `COD` (Codeunit), `PAG` (Page), `REP` (Report), `XML` (XMLport), `QUE` (Query), `MEN` (MenuSuite), `DAT` (Dataport — oud)

## Available Commands (7)

| Command | Purpose |
|---------|---------|
| `/cal-init` | Nieuw project: instance aanmaken, structuur, initiële export + CLAUDE.md |
| `/cal-pull` | Sync database → git: exporteer Modified=Yes objecten, toon diff |
| `/cal-export` | Exporteer specifieke C/AL objecten uit de database naar tekst |
| `/cal-import` | Importeer en compileer C/AL objecten in de database |
| `/cal-git` | Git-beheer voor C/AL: export-strategie, diffs, merges |
| `/cal-review` | Review C/AL code op kwaliteit, patronen en valkuilen |
| `/cal-deploy` | Deploy C/AL wijzigingen naar omgevingen via de runner |
