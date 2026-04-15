---
name: cal-init
description: Initialize a new C/AL project - set up git repo, configure BC instance, and do initial export of all custom objects
allowed-tools: Bash, Read, Write, Glob
---

# CAL Init

Initialiseer een nieuw C/AL project: maak de git-structuur aan, koppel de BC/NAV server instance aan de database, en exporteer alle custom objecten als startpunt.

## Input

$ARGUMENTS — verplicht of optioneel:
- `<database>` — naam van de NAV/BC database (verplicht)
- `<pad>` — doelmap voor het project (optioneel, default: `~/repos/<database>`)
- `--server <naam>` — SQL server naam (optioneel, default: `localhost`)
- `--instance <naam>` — bestaande NAV server instance (optioneel, anders wordt er een aangemaakt)
- `--license <pad>` — pad naar `.flf` licentiebestand (optioneel)
- `--range <van..tot>` — object ID-range om te exporteren (optioneel, default: `50000..99999`)

## Instructies

### Stap 0 — Laad kennis

```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-tools.md" -o -name "cal-git-patterns.md" 2>/dev/null | head -2
```

### Stap 1 — Controleer NavAdminTool en Model.Tools beschikbaarheid

```powershell
# Zoek NavAdminTool (beheer) en Model.Tools (export/import)
$navAdminTool = Get-ChildItem 'C:\Program Files\Microsoft Dynamics 365 Business Central\*\Service\NavAdminTool.ps1' |
    Sort-Object { [int]($_.FullName -replace '.*\\(\d+)\\.*','$1') } -Descending | Select-Object -First 1

$modelTools = Get-ChildItem 'C:\Program Files (x86)\Microsoft Dynamics 365 Business Central\*\RoleTailored Client\Microsoft.Dynamics.Nav.Model.Tools.psd1' |
    Sort-Object { [int]($_.FullName -replace '.*\\(\d+)\\.*','$1') } -Descending | Select-Object -First 1

$finsql = Get-ChildItem 'C:\Program Files (x86)\Microsoft Dynamics 365 Business Central\*\RoleTailored Client\finsql.exe' |
    Sort-Object { [int]($_.FullName -replace '.*\\(\d+)\\.*','$1') } -Descending | Select-Object -First 1
```

Gebruik de hoogst beschikbare versie. Meld als er geen tooling gevonden wordt.

### Stap 2 — Zoek of maak een server instance

**Als er al een instance is die naar de database wijst:**
```powershell
Import-Module $navAdminTool.FullName -WarningAction SilentlyContinue
Get-NAVServerInstance | ForEach-Object {
    $cfg = Get-NAVServerConfiguration -ServerInstance $_.ServerInstance
    $db = ($cfg | Where-Object Key -eq 'DatabaseName').Value
    if ($db -eq $dbName) { Write-Host "Instance gevonden: $($_.ServerInstance)" }
}
```

**Als er geen instance is — maak een nieuwe aan:**
```powershell
# Bepaal vrije poorten (check bestaande instances)
$usedPorts = Get-NAVServerInstance | ForEach-Object {
    $cfg = Get-NAVServerConfiguration -ServerInstance $_.ServerInstance
    $cfg | Where-Object { $_.Key -match 'Port' } | Select-Object -ExpandProperty Value
}

# Gebruik 9xxx range als 7xxx/8xxx bezet zijn
New-NAVServerInstance -ServerInstance "BC140-$dbName" `
    -DatabaseServer $dbServer `
    -DatabaseName $dbName `
    -ManagementServicesPort 7345 `
    -ClientServicesPort 7346 `
    -SOAPServicesPort 9047 `
    -ODataServicesPort 9048 `
    -Confirm:$false
```

**Poortconflict bij aanmaken?** Verhoog de poorten (+100) totdat ze vrij zijn.

### Stap 3 — Database online brengen (indien nodig)

Controleer of de database in RESTORING staat staat:
```powershell
# Via Windows services config — gebruik NavAdminTool, niet SQL
$state = (Get-NAVServerInstance -ServerInstance $instanceName).State
```

Als de service niet start controleer het Windows Event Log:
```powershell
Get-WinEvent -LogName 'Application' -MaxEvents 20 |
    Where-Object { $_.Message -match $instanceName } |
    Select-Object TimeCreated, LevelDisplayName, Message |
    Sort-Object TimeCreated -Descending | Select-Object -First 5
```

**Veelvoorkomende opstartfouten en oplossingen:**

| Fout in Event Log | Oorzaak | Oplossing |
|-------------------|---------|-----------|
| "database te recent" / "too recent" | DB gemaakt met iets nieuwere build | Zie hieronder: patch `databaseversionno` |
| "in the middle of a restore" | NORECOVERY restore onderbroken | `RESTORE DATABASE [naam] WITH RECOVERY` via sqlcmd |
| "port already registered" | Poortconflict met andere instance | Andere poorten kiezen in Stap 2, opnieuw starten |
| "cannot start the service" | Zie Event Log voor detail | Lees de volledige Event Log entry |

**Alleen bij fout "database te recent":** vergelijk het versieverschil en patch indien het klein is (zelfde major versie, andere CU-build):

```powershell
# Lees versie van een werkende BC-database op dezelfde server als referentie
# Via sqlcmd met script file (vermijdt $ escaping problemen)
$script = "SELECT databaseversionno, applicationversion FROM [dbo].[$" + "ndo$" + "dbproperty]"

# Referentie: werkende database
$ref = sqlcmd -S $dbServer -d '<werkende-db>' -E -No -Q $script

# Doel: de te initialiseren database
$target = sqlcmd -S $dbServer -d $dbName -E -No -Q $script
```

Verschil van 1–5 in `databaseversionno` en zelfde major versie → patch is veilig:
```sql
-- Sla op als .sql bestand, voer uit via sqlcmd -i
UPDATE [dbo].[$ndo$dbproperty]
SET databaseversionno = <referentie-waarde>,
    applicationversion = '<referentie-versie>'
```

> Nooit patchen bij major versie-upgrades (bijv. NAV2018 → BC14) — dan is een database-conversie nodig.

### Stap 4 — Start de instance en importeer licentie

```powershell
# Start service
Set-NAVServerInstance -ServerInstance $instanceName -Start
Start-Sleep -Seconds 10

# Controleer staat
$state = (Get-NAVServerInstance -ServerInstance $instanceName).State
Write-Host "Staat: $state"

# Importeer licentie als opgegeven
if ($licenseFile) {
    Import-NAVServerLicense -ServerInstance $instanceName `
        -LicenseFile $licenseFile -Database NavDatabase
}
```

Licentie zoeken als niet opgegeven:
```bash
find ~/repos -name "dev.flf" -o -name "*.flf" 2>/dev/null | grep -v Cronus | head -5
```

### Stap 5 — Maak projectstructuur aan

```bash
mkdir -p <projectpad>/{objects,reports,scripts}
cd <projectpad>
git init

cat > .gitignore << 'EOF'
*.fob
*.log
finsql.log
export_temp/
*.bak
EOF
```

### Stap 6 — Exporteer alle custom objecten

```powershell
Import-Module $modelTools.FullName -WarningAction SilentlyContinue

$cfg = Get-NAVServerConfiguration -ServerInstance $instanceName
$dbServer = ($cfg | Where-Object Key -eq 'DatabaseServer').Value
$dbName   = ($cfg | Where-Object Key -eq 'DatabaseName').Value

$typeMap = @{
    'Table'     = 'TAB'; 'Codeunit' = 'COD'; 'Page'      = 'PAG'
    'Report'    = 'REP'; 'XMLport'  = 'XML'; 'Query'     = 'QUE'
    'MenuSuite' = 'MEN'; 'Form'     = 'FOR'; 'Dataport'  = 'DAT'
}

# Export alles naar één tijdelijk bestand
$tmpAll = [System.IO.Path]::GetTempFileName() + '.txt'
Export-NAVApplicationObject `
    -DatabaseServer $dbServer -DatabaseName $dbName `
    -Filter "ID=$idRange" -Path $tmpAll -Force

# Split per object
$tmpDir = Join-Path ([System.IO.Path]::GetTempPath()) 'cal-init-split'
if (Test-Path $tmpDir) { Remove-Item $tmpDir -Recurse -Force }
New-Item -ItemType Directory -Path $tmpDir | Out-Null
Split-NAVApplicationObjectFile -Source $tmpAll -Destination $tmpDir

# Verplaats naar project, normaliseer datum/tijd
foreach ($file in Get-ChildItem $tmpDir -Filter '*.txt') {
    $content = Get-Content $file.FullName -Raw
    if ($content -match '^OBJECT (\w+) (\d+) (.+)') {
        $type = $Matches[1]; $id = $Matches[2]; $name = $Matches[3].Trim()
        $prefix = if ($typeMap[$type]) { $typeMap[$type] } else { $type.Substring(0,3).ToUpper() }

        # Normaliseer datum/tijd — voorkomt ruis in git diff
        $content = $content -replace 'Date=\d{2}-\d{2}-\d{2};', 'Date=;'
        $content = $content -replace 'Time=\d{2}:\d{2}:\d{2};', 'Time=;'

        $destDir  = if ($type -eq 'Report') { "$projectPath\reports" } else { "$projectPath\objects" }
        $destFile = Join-Path $destDir ($prefix + $id + ' - ' + $name + '.txt')
        [System.IO.File]::WriteAllText($destFile, $content, [System.Text.Encoding]::UTF8)
    }
}

# Opruimen
Remove-Item $tmpAll, $tmpDir -Recurse -Force
```

### Stap 7 — Maak CLAUDE.md aan

Maak `CLAUDE.md` in de projectmap met:

```markdown
# CLAUDE.md — <projectnaam>

## C/AL Database

- **Database**: `<databasenaam>` op `<server>`
- **BC Instance**: `<instancenaam>`
- **Versie**: BC14 / NAV 20xx
- **Object range**: 50000–99999

## Toolpaden

- NavAdminTool: `<pad naar NavAdminTool.ps1>`
- Model.Tools: `<pad naar Microsoft.Dynamics.Nav.Model.Tools.psd1>`
- finsql.exe: `<pad naar finsql.exe>`

## Omgevingen

| Naam | Instance | SyncMode |
|------|----------|----------|
| Dev  | <instancenaam> | ForceSync |

## Workflow

1. `/cal-pull` — haal laatste code op uit database
2. Bewerk in VSCode (installeer `zodiacfireworks.vscode-c-al` voor syntax highlighting)
3. `/cal-review` — controleer vóór import
4. `/cal-import` — compileer in DB
5. Test in client
6. `/cal-git commit` — commit naar git
7. `/cal-deploy` — deploy naar andere omgevingen
```

### Stap 8 — Initieel commit

```bash
cd <projectpad>
git add .
git commit -m "feat: initiële export van alle C/AL objecten uit <database>

<N> objecten geëxporteerd via <instance>:
- <X> tabellen (TAB)
- <X> codeunits (COD)
- <X> pagina's (PAG)
- <X> overige

Database: <database> (<server>)"
```

## Rapporteer

Na voltooiing:
- Instance naam en staat (Running/Stopped)
- Aantal geëxporteerde objecten per type
- Projectpad
- Eventuele waarschuwingen (gepatchte versie, ontbrekende licentie)

## Regels

- **Vraag bevestiging** voordat je een versie-patch uitvoert — dit is onomkeerbaar zonder backup
- **Maak nooit een nieuwe instance** als er al één is voor deze database — zoek altijd eerst
- **Normaliseer altijd datum/tijd** — anders is elke export een diff-storm
- **Één commit** voor de initiële export — niet per object
