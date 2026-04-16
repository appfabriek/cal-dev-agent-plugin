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
| Compile-problemen, tenant Failed, versie-mismatch | `cal-compile-troubleshooting.md` |
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
| "maak aanpassing in X" / "voeg Y toe" / "wijzig Z" | `/cal-dev` — complete workflow incl. compile en test |
| "importeer deze wijziging" | `Import-NAVApplicationObject` via NavAdminTool |
| "wat zijn de verschillen met productie?" | Export beide → git diff |
| "compileer object X" | `/cal-dev` of `Import-NAVApplicationObject` met NavServer params |
| "welke versie van object X draait?" | `Get-NAVApplicationObjectProperty` |
| "compile werkt niet" / "tenant Failed" / "503" | `cal-compile-troubleshooting.md` lezen |

## Development Workflow

### Scenario 1 — Bestaande omgeving

1. **Pre-conditie check** — service Running? tenant Operational? Developer Services bereikbaar (niet 503)?
2. **Export eerst** — haal het object op uit de **database** vóór je wijzigingen maakt (niet uit git)
3. **Bewerk in tekst** — `.txt` formaat is diffbaar en trackbaar in git
4. **Valideer syntax** — C/AL heeft strikte whitespace-regels in het exportformaat
5. **Importeer MET NavServer params** — zonder NavServer wordt Object Metadata niet geschreven → object niet gecompileerd
6. **Controleer Compiled=Yes** — exporteer het object opnieuw en check de status
7. **Test via Invoke-NAVCodeunit** — altijd testen na compile
8. **Git commit** — exporteer de definitieve versie uit DB, normaliseer datum/tijd, dan commit
9. **Oplevering als .fob** — exporteer compiled object als .fob voor overdracht

### Scenario 2 — Docker container

Gebruik als er geen passende omgeving beschikbaar is. Zie `/cal-dev` skill voor volledige instructies.

Snelle workflow:
```powershell
Import-Module BcContainerHelper
$containerName = 'cal-dev-test-bc14'
$artifactUrl = Get-BcArtifactUrl -type OnPrem -version '14' -country nl
New-BcContainer -accept_eula -accept_outdated -containerName $containerName `
    -artifactUrl $artifactUrl -auth NavUserPassword `
    -credential (New-Object PSCredential('admin', (ConvertTo-SecureString 'P@ssw0rd1!' -AsPlainText -Force))) `
    -isolation hyperv -memoryLimit '4G' -licenseFile 'D:\repos\plants\license\BC14_DEV_NL.flf'
```

**Beschikbare versies als Docker artifact:**
- BC14 (14.x) — volledig getest ✓
- BC13 (13.x) — getest ✓
- NAV 2018 (11.x), NAV 2017 (10.x), NAV 2016 (9.x) — demo licentie verlopen, gebruik altijd `-licenseFile`
- NAV 2013/R2 — NIET beschikbaar als Docker artifact

**Kritieke valkuilen Docker:**
- Gebruik ALTIJD `-licenseFile` bij aanmaken van de container (of vervang Cronus.flf achteraf)
- `docker exec`/`docker cp` werkt NIET met Hyper-V isolatie — gebruik `Invoke-ScriptInBcContainer`
- Import via finsql.exe ZONDER NavServer params (directe SQL), dan compile MET NavServer
- Omgevingsvariabelen zijn leeg — laad via `. 'c:\run\ServiceSettings.ps1'`

Zie `cal-compile-troubleshooting.md` (secties 12 en 13) voor Docker-specifieke problemen.

> **Kritieke valkuil (algemeen):** `Import-NAVApplicationObject` zonder NavServer params importeert de broncode maar compileert NIET (Object Metadata wordt niet geschreven). Gebruik altijd `-NavServerName`, `-NavServerInstance`, `-NavServerManagementPort`.

> **Kritieke valkuil (algemeen):** `finsql.exe` met `usesyntheticmessages=yes` werkt NIET op Nederlandse BC. Gebruik altijd PowerShell cmdlets (of directe finsql zonder dat flag).

Zie `cal-compile-troubleshooting.md` bij problemen.

## Objectnaamgeving in git

Gebruik de conventie: `<type><id> - <naam>.txt`

Voorbeelden:
- `TAB50000 - Customer Extension.txt`
- `COD50000 - My Codeunit.txt`
- `PAG50000 - Customer Card Extension.txt`
- `REP50000 - My Report.txt`

Object type afkortingen: `TAB` (Table), `COD` (Codeunit), `PAG` (Page), `REP` (Report), `XML` (XMLport), `QUE` (Query), `MEN` (MenuSuite), `DAT` (Dataport — oud)

## Available Commands (8)

| Command | Purpose |
|---------|---------|
| `/cal-init` | Nieuw project: instance aanmaken, structuur, initiële export + CLAUDE.md |
| `/cal-pull` | Sync database → git: exporteer Modified=Yes objecten, toon diff |
| `/cal-export` | Exporteer specifieke C/AL objecten uit de database naar tekst |
| `/cal-import` | Importeer en compileer C/AL objecten in de database |
| `/cal-dev` | **Complete dev workflow**: code ophalen, aanpassen, compileren, testen, opleveren als .fob |
| `/cal-git` | Git-beheer voor C/AL: export-strategie, diffs, merges |
| `/cal-review` | Review C/AL code op kwaliteit, patronen en valkuilen |
| `/cal-deploy` | Deploy C/AL wijzigingen naar omgevingen via de runner |
