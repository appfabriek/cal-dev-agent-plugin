---
name: cal-pull
description: Pull latest C/AL code from database into git - sync modified objects from DB to local files
allowed-tools: Bash, Read, Write, Glob
---

# CAL Pull

Haal de laatste C/AL code op uit de database en synchroniseer met de lokale git repository. Bedoeld voor situaties waarbij wijzigingen direct in de database zijn gemaakt (via C/SIDE client) en nog niet in git staan.

## Input

$ARGUMENTS — optionele instructies:
- (leeg) — exporteer alle objecten met `Modified=Yes` vlag in de database
- "alle" — exporteer alle custom objecten (50000+), ongeacht Modified-vlag
- "TAB" of "COD50000" — specifiek type of object
- "check" — rapporteer welke objecten in DB gewijzigd zijn t.o.v. git, wijzig niets

## Wanneer gebruiken

- Na C/SIDE-sessie waarbij objecten direct in de database zijn gewijzigd
- Bij overname van een database van een klant die nooit git gebruikte
- Bij sync na deployment vanuit een andere omgeving
- Regelmatig bij klanten die code-wijzigingen in de database laten staan

## Instructies

### Stap 0 — Laad kennis

```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-tools.md" -o -name "cal-git-patterns.md" 2>/dev/null | head -2
```

Lees ook `CLAUDE.md` van het huidige project voor instance-naam en databasegegevens.

### Stap 1 — Bepaal scope

**Gewijzigde objecten detecteren (Modified=Yes):**
```powershell
Import-Module $navAdminTool -WarningAction SilentlyContinue
Import-Module $modelTools   -WarningAction SilentlyContinue

$cfg      = Get-NAVServerConfiguration -ServerInstance $env:BC_INSTANCE
$dbServer = ($cfg | Where-Object Key -eq 'DatabaseServer').Value
$dbName   = ($cfg | Where-Object Key -eq 'DatabaseName').Value

# Objecten met Modified=Yes in de database
$modified = @()
$tmpCheck = [System.IO.Path]::GetTempFileName() + '.txt'

Export-NAVApplicationObject `
    -DatabaseServer $dbServer -DatabaseName $dbName `
    -Filter 'Modified=Yes;ID=50000..99999' -Path $tmpCheck -Force

if ((Get-Item $tmpCheck).Length -gt 0) {
    Split-NAVApplicationObjectFile -Source $tmpCheck -Destination ($tmpCheck + '_split')
    $modified = Get-ChildItem ($tmpCheck + '_split') -Filter '*.txt'
    Write-Host "Gewijzigd in DB (Modified=Yes): $($modified.Count) objecten"
} else {
    Write-Host "Geen objecten met Modified=Yes gevonden."
}
Remove-Item $tmpCheck -Force -ErrorAction SilentlyContinue
```

**Vergelijk ook met git:**
```bash
# Welke bestanden zijn in git gewijzigd maar nog niet in DB gesynchroniseerd?
git diff --name-only HEAD objects/ reports/

# Welke objecten bestaan in DB maar niet in git?
# (ontbreekt na cal-init, of nieuwe objecten toegevoegd in C/SIDE)
```

### Stap 2 — Export scope bepalen

| Situatie | Filter |
|----------|--------|
| Dagelijkse sync (gewijzigd) | `Modified=Yes;ID=50000..99999` |
| Alles ophalen | `ID=50000..99999` |
| Specifiek type | `Type=Codeunit;ID=50000..99999` |
| Specifiek object | `Type=Codeunit;ID=50000` |

### Stap 3 — Exporteer en normaliseer

```powershell
$typeMap = @{
    'Table'     = 'TAB'; 'Codeunit' = 'COD'; 'Page'      = 'PAG'
    'Report'    = 'REP'; 'XMLport'  = 'XML'; 'Query'     = 'QUE'
    'MenuSuite' = 'MEN'; 'Form'     = 'FOR'; 'Dataport'  = 'DAT'
}

$tmpAll = [System.IO.Path]::GetTempFileName() + '.txt'
Export-NAVApplicationObject `
    -DatabaseServer $dbServer -DatabaseName $dbName `
    -Filter $filter -Path $tmpAll -Force

$tmpDir = Join-Path ([System.IO.Path]::GetTempPath()) 'cal-pull-split'
if (Test-Path $tmpDir) { Remove-Item $tmpDir -Recurse -Force }
New-Item -ItemType Directory -Path $tmpDir | Out-Null
Split-NAVApplicationObjectFile -Source $tmpAll -Destination $tmpDir

$updated = 0; $new = 0

foreach ($file in Get-ChildItem $tmpDir -Filter '*.txt') {
    $content = Get-Content $file.FullName -Raw
    if ($content -match '^OBJECT (\w+) (\d+) (.+)') {
        $type = $Matches[1]; $id = $Matches[2]; $name = $Matches[3].Trim()
        $prefix = if ($typeMap[$type]) { $typeMap[$type] } else { $type.Substring(0,3).ToUpper() }

        # Normaliseer datum/tijd
        $content = $content -replace 'Date=\d{2}-\d{2}-\d{2};', 'Date=;'
        $content = $content -replace 'Time=\d{2}:\d{2}:\d{2};', 'Time=;'

        $destDir  = if ($type -eq 'Report') { 'reports' } else { 'objects' }
        $destFile = Join-Path $destDir ($prefix + $id + ' - ' + $name + '.txt')

        $isNew = -not (Test-Path $destFile)
        [System.IO.File]::WriteAllText($destFile, $content, [System.Text.Encoding]::UTF8)
        if ($isNew) { $new++ } else { $updated++ }
    }
}

Remove-Item $tmpAll, $tmpDir -Recurse -Force
Write-Host "Bijgewerkt: $updated  Nieuw: $new"
```

### Stap 4 — Reset Modified-vlag in database

Na succesvolle export de Modified-vlag resetten zodat de database weer gesynchroniseerd is:

```powershell
# Reset Modified=Yes op geëxporteerde objecten
# Gebruik Set-NAVApplicationObjectProperty per object
foreach ($obj in $exportedObjects) {
    Set-NAVApplicationObjectProperty `
        -TargetPath $obj.FullName `
        -ModifiedProperty No
    # Of via Get-NAVApplicationObjectProperty om te verifiëren
}
```

> **Let op:** Reset de Modified-vlag alleen als je zeker weet dat de export geslaagd is en de bestanden correct in git staan.

### Stap 5 — Toon git diff en rapporteer

```bash
# Wat is er veranderd?
git diff --stat objects/ reports/

# Gedetailleerde diff per object
git diff objects/
```

Interpreteer de diff:
- `+  { <veldnummer> ;` → nieuw veld toegevoegd (SyncMode=Yes bij volgende import elders)
- `-  { <veldnummer> ;` → veld verwijderd (SyncMode=Force bij volgende import elders!)
- Alleen CODE-wijzigingen → SyncMode=No

Rapporteer:
- Hoeveel objecten bijgewerkt / nieuw
- Welke schema-wijzigingen gedetecteerd (velden toegevoegd/verwijderd)
- Aanbevolen SyncMode voor deployment naar andere omgevingen

### Stap 6 — Optioneel committen

Bij `cal-pull commit` of als gebruiker vraagt te committen:

```bash
git add objects/ reports/
git commit -m "sync: haal laatste C/AL code op uit database

Gewijzigd in DB (Modified=Yes):
- <lijst van objecten>

Schema-wijzigingen: <ja/nee — SyncMode vereist>
Database: <databasenaam> (<server>)"
```

## Regels

- **Nooit blindelings overschrijven** — toon altijd eerst de diff, laat de gebruiker bevestigen bij schema-wijzigingen
- **Normaliseer altijd datum/tijd** — anders zijn elke export ruis in de diff
- **Reset Modified-vlag pas na commit** — niet eerder, anders verlies je het spoor
- **Detecteer schema-wijzigingen** — waarschuw expliciet bij verwijderde velden (SyncMode=Force nodig bij deployment)
- **Nieuwe objecten** apart melden — objecten die nog niet in git stonden zijn potentieel vergeten wijzigingen
