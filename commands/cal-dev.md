---
name: cal-dev
description: Complete C/AL development workflow - code ophalen, aanpassen, compileren, testen en opleveren als .fob
allowed-tools: Bash, Read, Write, Edit, Glob
---

# CAL Dev

Complete workflow voor een C/AL aanpassing: van huidige code ophalen tot oplevering als .fob met wijzigingsomschrijving.

## Input

$ARGUMENTS — wat je wil doen:
- "COD50000 voeg procedure X toe" — object + beschrijving van de wijziging
- "TAB50000 voeg veld is_foodtruck Boolean toe" — tabel met veldwijziging
- (leeg) — vraag de gebruiker wat er aangepast moet worden

---

## Instructies

### Stap 0 — Laad kennis

Lees vóór alles:
```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-tools.md" 2>/dev/null | head -1
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-compile-troubleshooting.md" 2>/dev/null | head -1
```

Lees ook de project `CLAUDE.md` voor instance-naam en databasegegevens.

---

### Stap 1 — Pre-conditie check

Controleer ALTIJD de omgeving vóór je begint. Sla niets over.

```powershell
$navAdmin   = 'C:\Program Files\Microsoft Dynamics 365 Business Central\140\Service\NavAdminTool.ps1'
$modelTools = 'C:\Program Files (x86)\Microsoft Dynamics 365 Business Central\140\RoleTailored Client\Microsoft.Dynamics.Nav.Model.Tools.psd1'
Import-Module $navAdmin   -WarningAction SilentlyContinue
Import-Module $modelTools -WarningAction SilentlyContinue

# Instance naam ophalen uit project CLAUDE.md of gebruiker vragen
$inst = 'BC140-KMT'  # pas aan

# 1. Service running?
$svc = Get-NAVServerInstance -ServerInstance $inst
Write-Host "Service: $($svc.State) — versie $($svc.Version)"
if ($svc.State -ne 'Running') { throw "Service is niet Running" }

# 2. Tenant Operational?
$tenant = Get-NAVTenant -ServerInstance $inst -Tenant default -ErrorAction SilentlyContinue
Write-Host "Tenant: $($tenant.State)"
if ($tenant.State -match 'Failed') {
    Write-Host "DetailedState: $($tenant.DetailedState)"
    throw "Tenant is Failed — zie cal-compile-troubleshooting.md voor diagnose"
}

# 3. Database naam (NOOIT hardcoden — altijd uit config)
$dbServer = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DatabaseServer').Value
$dbName   = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DatabaseName').Value
Write-Host "Database: $dbServer\$dbName"

# 4. Developer Services bereikbaar?
$devPort = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DeveloperServicesPort').Value
try {
    Invoke-WebRequest "http://localhost:$devPort/$inst/" -TimeoutSec 5 -UseBasicParsing | Out-Null
    Write-Host "Developer Services: OK (poort $devPort)"
} catch {
    if ($_ -match '503') {
        Write-Host "Developer Services: 503 — tenant niet initialiseerbaar, schema sync zal mislukken"
        Write-Host "Zie cal-compile-troubleshooting.md sectie 'Developer Services 503'"
    } elseif ($_ -match '401|403') {
        Write-Host "Developer Services: bereikbaar (auth vereist — OK)"
    } else {
        Write-Host "Developer Services: $($_.Exception.Message)"
    }
}
```

**Stop als:**
- Service is niet Running
- Tenant is Failed (los dit eerst op via troubleshooting guide)
- Database naam onbekend

---

### Stap 2 — Huidige code ophalen

Exporteer het te wijzigen object uit de database. Dit is de **actuele** versie, niet wat in git staat.

```powershell
# Bepaal type en ID uit $ARGUMENTS
# Voorbeelden: Type=Codeunit;ID=50000 / Type=Table;ID=50000

$filter   = "Type=Codeunit;ID=50000"   # pas aan
$outFile  = "C:\temp\COD50000-current.txt"

Export-NAVApplicationObject `
    -DatabaseServer $dbServer `
    -DatabaseName   $dbName `
    -Filter         $filter `
    -Path           $outFile `
    -Force

$props = Get-NAVApplicationObjectProperty -Source $outFile
Write-Host "Geëxporteerd: $($props.Type) $($props.Id) — $($props.Name)"
Write-Host "Compiled=$($props.Compiled)  Modified=$($props.Modified)  VersionList=$($props.VersionList)"
```

Vergelijk ook met de git-versie:
```bash
git diff objects/COD50000\ -\ *.txt -- 2>/dev/null || echo "(nog niet in git)"
```

> **Let op:** Gebruik altijd de export uit de database als basis, niet alleen wat in git staat. De database is de bron van waarheid voor C/AL.

---

### Stap 3 — Aanpassing maken

Bewerk het geëxporteerde `.txt` bestand.

**C/AL object formaat regels:**
- `OBJECT-PROPERTIES` blok: `Date=;` en `Time=;` leeglaten (normaliseer altijd)
- `Modified=Yes;` → wordt automatisch gezet bij import, niet handmatig aanpassen
- Velden in een tabel: `{ <nr> ; <naam> ; <type> ; ... }` formaat
- Procedures: `PROCEDURE <nr><naam>@<id>();` met `BEGIN ... END;`
- Globale variabelen direct na de proceduredefinitie in `VAR ... END;`

**Bij tabel-wijzigingen (nieuw veld):**
- Voeg het veld toe in het `FIELDS` blok
- Controleer ook triggers op het veld (`OnValidate`, etc.)
- Tabel objecten vereisen schema sync → `SynchronizeSchemaChanges=Force`

**Bij codeunit/pagina-wijzigingen (alleen code):**
- `SynchronizeSchemaChanges=No` volstaat

---

### Stap 4 — Import en compile

> **KRITIEK:** Gebruik `Import-NAVApplicationObject` MET NavServer parameters. Zonder NavServer params wordt het object geïmporteerd maar **niet gecompileerd** (Object Metadata wordt niet geschreven).

```powershell
$ConfirmPreference = 'None'

# Bepaal SynchronizeSchemaChanges:
# - 'No'    → alleen code-wijzigingen (codeunit, page, report)
# - 'Yes'   → nieuwe velden in tabel
# - 'Force' → destructieve wijzigingen (veld verwijderen, type wijzigen, of onzeker)
$syncMode = 'Force'  # voor tabellen altijd Force, dan weet je het zeker

$importFile = "C:\temp\COD50000-current.txt"  # het aangepaste bestand

Import-NAVApplicationObject `
    -Path                      $importFile `
    -DatabaseServer            $dbServer `
    -DatabaseName              $dbName `
    -NavServerName             'localhost' `
    -NavServerInstance         $inst `
    -NavServerManagementPort   7345 `
    -ImportAction              Overwrite `
    -SynchronizeSchemaChanges  $syncMode `
    -Confirm:$false
```

**Controleer compilatiestatus direct na import:**
```powershell
$checkFile = "C:\temp\check-compiled.txt"
Export-NAVApplicationObject -DatabaseServer $dbServer -DatabaseName $dbName `
    -Filter $filter -Path $checkFile -Force

$compiled = Get-NAVApplicationObjectProperty -Source $checkFile
Write-Host "Compiled=$($compiled.Compiled)  Modified=$($compiled.Modified)"

if (-not ($compiled.Compiled -match '1|Yes|True')) {
    Write-Host "WAARSCHUWING: Object is niet gecompileerd!"
    Write-Host "Zie cal-compile-troubleshooting.md voor oplossingen."
}
```

---

### Stap 5 — Test

Test de wijziging via `Invoke-NAVCodeunit`. Als er geen apart test-codeunit is, verifieer minimaal via de event log.

**Optie A: Apart test-codeunit (aanbevolen)**
```powershell
# Controleer of de huidige gebruiker SUPER rechten heeft
$company = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'NASServicesStartupArgument').Value -replace 'company=',''
if (-not $company) { $company = 'CRONUS' }  # fallback

try {
    Invoke-NAVCodeunit `
        -ServerInstance $inst `
        -CompanyName    $company `
        -CodeunitId     50031 `
        -ErrorAction    Stop
    Write-Host "Test GESLAAGD"
} catch {
    if ($_ -match 'Access is denied') {
        Write-Host "Permissie-fout — voeg SUPER toe voor huidige gebruiker:"
        Write-Host "New-NAVServerUserPermissionSet -ServerInstance $inst -WindowsAccount '$env:USERDOMAIN\$env:USERNAME' -PermissionSetId SUPER -CompanyName '$company'"
    } else {
        Write-Host "Test MISLUKT: $_"
    }
}
```

**Optie B: Event log check**
```powershell
Get-EventLog -LogName Application -Source '*Dynamics*' -Newest 5 -EntryType Error,Warning `
    -After (Get-Date).AddMinutes(-2) |
    Select-Object TimeGenerated, EntryType, @{N='Msg';E={$_.Message.Substring(0,[Math]::Min(300,$_.Message.Length))}} |
    Format-List
```

---

### Stap 6 — Opslaan in git

Exporteer de definitieve versie uit de database (de gecompileerde versie) naar de git repository:

```powershell
# Exporteer naar objects/ map
$gitDir  = "D:\repos\kmt\objects"  # pas aan naar project
$objName = "COD50000 - MijnCodeunit.txt"  # pas aan

Export-NAVApplicationObject `
    -DatabaseServer $dbServer `
    -DatabaseName   $dbName `
    -Filter         $filter `
    -Path           (Join-Path $gitDir $objName) `
    -Force

# Normaliseer datum/tijd zodat git diff schoon is
$content = Get-Content (Join-Path $gitDir $objName) -Raw -Encoding Default
$content = $content -replace 'Date=\d{2}-\d{2}-\d{2};', 'Date=;'
$content = $content -replace 'Time=\d{2}:\d{2}:\d{2};', 'Time=;'
Set-Content -Path (Join-Path $gitDir $objName) -Value $content -Encoding Default -NoNewline
```

```bash
git add objects/COD50000\ -\ MijnCodeunit.txt
git commit -m "feat(mijn-module): beschrijving van de wijziging (#issuenr)"
```

---

### Stap 7 — Oplevering (.fob exporteren)

Exporteer het gecompileerde object als `.fob` voor overdracht naar andere omgevingen:

```powershell
$fobPath = "C:\temp\COD50000-v1.0.fob"

Export-NAVApplicationObject `
    -DatabaseServer    $dbServer `
    -DatabaseName      $dbName `
    -Filter            $filter `
    -Path              $fobPath `
    -Force

Write-Host "FOB aangemaakt: $fobPath"
Write-Host "Grootte: $((Get-Item $fobPath).Length) bytes"
```

**Maak ook een wijzigingsomschrijving:**

Rapporteer aan de gebruiker:
- Object type + ID + naam
- Wat er gewijzigd is (op basis van git diff)
- SynchronizeSchemaChanges mode die nodig is bij deployment naar andere omgevingen
- Eventuele bijzonderheden (schema-wijzigingen, nieuwe permissies, etc.)

Voorbeeld:
```
Opgeleverd: COD50000 - MijnCodeunit
Wijziging: procedure fVoegActieToe toegevoegd (alleen code, geen schema-wijziging)
Deployment: SynchronizeSchemaChanges=No
FOB: C:\temp\COD50000-v1.0.fob
Git: objects/COD50000 - MijnCodeunit.txt (commit abc1234)
```

---

## Scenario 2 — Docker container omgeving

Gebruik dit scenario als **er geen bestaande BC/NAV omgeving beschikbaar is** die overeenkomt met de doelversie, of als de bestaande omgeving te ver afwijkt om op te compileren.

> **Vereiste tools:** Docker Desktop (met Hyper-V), BcContainerHelper PowerShell module, developer license (.flf)

### Stap A — Container aanmaken

```powershell
Import-Module BcContainerHelper -WarningAction SilentlyContinue

$containerName = 'cal-dev-test-bc14'
$credential    = New-Object PSCredential('admin', (ConvertTo-SecureString 'P@ssw0rd1!' -AsPlainText -Force))
$licFile       = 'D:\repos\plants\license\BC14_DEV_NL.flf'  # VEREIST - zie pitfalls

# Bepaal artifact URL per versie:
# BC14 CU53:  Get-BcArtifactUrl -type OnPrem -version '14.53' -country nl
# BC14 latest: Get-BcArtifactUrl -type OnPrem -version '14' -country nl
# BC13:        Get-BcArtifactUrl -type OnPrem -version '13' -country nl
# NAV 2018:    Get-BcArtifactUrl -type OnPrem -version '11' -country nl
# NAV 2017:    Get-BcArtifactUrl -type OnPrem -version '10' -country nl
# NAV 2016:    Get-BcArtifactUrl -type OnPrem -version '9'  -country nl
# NAV 2013/R2: NIET BESCHIKBAAR als Docker artifact

$artifactUrl = Get-BcArtifactUrl -type OnPrem -version '14' -country nl
Write-Host "Artifact: $artifactUrl"

New-BcContainer `
    -accept_eula `
    -accept_outdated `
    -containerName  $containerName `
    -artifactUrl    $artifactUrl `
    -auth           NavUserPassword `
    -credential     $credential `
    -updateHosts `
    -memoryLimit    '4G' `
    -isolation      hyperv `
    -licenseFile    $licFile `
    -enableTaskScheduler:$false
```

**Container naam max 15 tekens** (BcContainerHelper geeft waarschuwing bij overschrijding).

> **Versie-ondersteuning (getest):**
> - BC14 ✓ (v14.x) — BC14 developer licentie vereist
> - BC13 ✓ (v13.x) — BC14 developer licentie vereist
> - NAV 2018 (v11), NAV 2017 (v10), NAV 2016 (v9) — CRONUS demo licentie verlopen + BC14 licentie NIET compatibel → vereist versie-specifieke NAV licentie
> - NAV 2013/R2 — **NIET beschikbaar** als Docker artifact

### Stap B — License vervangen (alleen indien NODIG)

Als je `-licenseFile $licFile` hebt doorgegeven bij `New-BcContainer` (aanbevolen), importeert BcContainerHelper de licentie al tijdens container-initialisatie — je kunt Stap B overslaan.

**Stap B is alleen nodig als de container aangemaakt is zonder `-licenseFile`.**
BcContainerHelper containers zonder licentie gebruiken de CRONUS demo licentie (7 KB). Deze licentie **ontbreekt "File, Import, Text" systeem-permissie** — import van objecten mislukt met fout `[18023703]`.

```powershell
# Vervang CRONUS demo licentie door echte developer licentie
$licContent = [System.IO.File]::ReadAllBytes($licFile)

Invoke-ScriptInBcContainer -containerName $containerName -scriptblock {
    param($licBytes)

    . 'c:\run\ServiceSettings.ps1'

    # Zoek en vervang Cronus.flf (versie-onafhankelijk pad)
    $cronusFlf = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV' -Recurse -Filter 'Cronus.flf' -ErrorAction SilentlyContinue |
        Select-Object -First 1 -ExpandProperty FullName
    if (-not $cronusFlf) {
        $cronusFlf = Get-ChildItem 'C:\Program Files\Microsoft Dynamics 365 Business Central' -Recurse -Filter 'Cronus.flf' -ErrorAction SilentlyContinue |
            Select-Object -First 1 -ExpandProperty FullName
    }

    [System.IO.File]::WriteAllBytes($cronusFlf, $licBytes)
    Write-Host "Licentie vervangen: $cronusFlf ($($licBytes.Length) bytes)"

    # Herstart service
    Restart-Service "MicrosoftDynamicsNavServer`$$ServerInstance" -Force
    Start-Sleep -Seconds 10
    Write-Host "Service: $((Get-Service "MicrosoftDynamicsNavServer`$$ServerInstance").Status)"

} -argumentList (,$licContent)
```

> **Belangrijk:** `Import-NAVServerLicense` werkt NIET voor containers — het schrijft naar de database maar de service laadt nog steeds Cronus.flf van disk. Vervang het bestand op disk en herstart.

### Stap C — Rechten instellen

```powershell
Invoke-ScriptInBcContainer -containerName $containerName -scriptblock {
    . 'c:\run\ServiceSettings.ps1'
    $dll = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV','C:\Program Files\Microsoft Dynamics 365 Business Central' -Recurse -Filter 'Microsoft.Dynamics.Nav.Management.dll' -ErrorAction SilentlyContinue |
        Select-Object -First 1 -ExpandProperty FullName
    Import-Module $dll -ErrorAction SilentlyContinue -WarningAction SilentlyContinue

    $company = (Get-NAVCompany -ServerInstance $ServerInstance | Select-Object -First 1).CompanyName
    $me = 'user manager\containeradministrator'

    try {
        New-NAVServerUser -ServerInstance $ServerInstance -WindowsAccount $me -LicenseType Full -ErrorAction Stop
        New-NAVServerUserPermissionSet -ServerInstance $ServerInstance -WindowsAccount $me -PermissionSetId 'SUPER' -CompanyName $company
        Write-Host "SUPER rechten: OK voor $me"
    } catch {
        Write-Host "Rechten: $($_.Exception.Message)"  # kan falen als user al bestaat
    }
}
```

> **Licentielimiet:** CRONUS demo licentie staat 2 full users toe (ADMIN + 1 extra). Voeg nooit meer dan 1 extra Windows user toe.

### Stap D — Objecten importeren (finsql.exe direct)

In containers gebruik je `Import-NAVApplicationObject` **via Invoke-ScriptInBcContainer**. Dit roept intern finsql.exe aan.

**Kritiek:** Gebruik finsql.exe ZONDER NavServer parameters voor de import, dan `Compile-NAVApplicationObject` MET NavServer voor compilatie.

```powershell
Invoke-ScriptInBcContainer -containerName $containerName -scriptblock {
    param($objectStr)

    . 'c:\run\ServiceSettings.ps1'
    $dll = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV','C:\Program Files\Microsoft Dynamics 365 Business Central' -Recurse -Filter 'Microsoft.Dynamics.Nav.Management.dll' -ErrorAction SilentlyContinue |
        Select-Object -First 1 -ExpandProperty FullName
    Import-Module $dll -ErrorAction SilentlyContinue -WarningAction SilentlyContinue

    $finsql = Get-ChildItem 'C:\Program Files (x86)\Microsoft Dynamics NAV','C:\Program Files (x86)\Microsoft Dynamics 365 Business Central' -Recurse -Filter 'finsql.exe' -ErrorAction SilentlyContinue |
        Select-Object -First 1 -ExpandProperty FullName

    $configPath = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV','C:\Program Files\Microsoft Dynamics 365 Business Central' -Recurse -Filter 'CustomSettings.config' -ErrorAction SilentlyContinue |
        Select-Object -First 1 -ExpandProperty FullName
    $raw   = Get-Content $configPath -Raw
    $db    = if ($raw -match 'key="DatabaseName"\s+value="([^"]*)"') { $Matches[1] } else { 'CRONUS' }
    $port  = if ($raw -match 'key="ManagementServicesPort"\s+value="([^"]*)"') { [int]$Matches[1] } else { 7045 }
    $dbSrv = 'localhost\SQLEXPRESS'

    New-Item -ItemType Directory -Path 'C:\temp' -Force | Out-Null
    $objPath = 'C:\temp\import-object.txt'
    $logPath = 'C:\temp\finsql-import.log'

    # Gebruik Windows-1252 encoding (finsql.exe verwacht dit)
    $enc = [System.Text.Encoding]::GetEncoding(1252)
    [System.IO.File]::WriteAllText($objPath, $objectStr, $enc)
    if (Test-Path $logPath) { Remove-Item $logPath -Force }

    # Import: finsql.exe ZONDER NavServer params (directe SQL write)
    $proc = Start-Process -FilePath $finsql `
        -ArgumentList "command=importobjects,file=`"$objPath`",logfile=`"$logPath`",servername=`"$dbSrv`",database=`"$db`",ntauthentication=yes" `
        -Wait -PassThru -NoNewWindow
    Write-Host "finsql import exit: $($proc.ExitCode)"
    if ((Test-Path $logPath) -and (Get-Item $logPath).Length -gt 0) {
        Write-Host "Log: $(Get-Content $logPath)"  # gevuld = FOUT
    }

    # Compileer MET NavServer (schema sync + Object Metadata schrijven)
    Compile-NAVApplicationObject `
        -DatabaseServer           $dbSrv `
        -DatabaseName             $db `
        -Filter                   'Type=Table;ID=50099'  `  # pas filter aan
        -NavServerName            'localhost' `
        -NavServerInstance        $ServerInstance `
        -NavServerManagementPort  $port `
        -SynchronizeSchemaChanges Force `
        -Recompile

    # Verificeer
    $r = Invoke-Sqlcmd -ServerInstance $dbSrv -Database $db `
        -Query "SELECT [Type], [ID], [Name], [Compiled] FROM [Object] WHERE [ID] = 50099"
    $r | ForEach-Object { Write-Host "  Type=$($_.Type) ID=$($_.ID) Compiled=$($_.Compiled)" }

} -argumentList $objectStr
```

### Stap E — Testen via Invoke-NAVCodeunit

```powershell
Invoke-ScriptInBcContainer -containerName $containerName -scriptblock {
    . 'c:\run\ServiceSettings.ps1'
    $dll = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV','C:\Program Files\Microsoft Dynamics 365 Business Central' -Recurse -Filter 'Microsoft.Dynamics.Nav.Management.dll' -ErrorAction SilentlyContinue |
        Select-Object -First 1 -ExpandProperty FullName
    Import-Module $dll -ErrorAction SilentlyContinue -WarningAction SilentlyContinue

    $company = (Get-NAVCompany -ServerInstance $ServerInstance | Select-Object -First 1).CompanyName
    Write-Host "Company: $company"

    try {
        Invoke-NAVCodeunit -ServerInstance $ServerInstance -CompanyName $company -CodeunitId 50099
        Write-Host "TEST GESLAAGD!"
    } catch {
        Write-Host "Fout: $($_.Exception.Message)"
    }
}
```

### Container versie-specifieke paden

Gebruik altijd `Get-ChildItem -Recurse` om paden te zoeken (versie-onafhankelijk). Onderstaande paden zijn **exact getest**:

| Versie | finsql.exe (getest) | Service base |
|--------|---------------------|--------------|
| BC14 ✓ | `C:\Program Files (x86)\Microsoft Dynamics NAV\140\RoleTailored Client\finsql.exe` | `C:\Program Files\Microsoft Dynamics NAV\140\Service\` |
| BC13 ✓ | `C:\Program Files (x86)\Microsoft Dynamics NAV\130\RoleTailored Client\finsql.exe` | `C:\Program Files\Microsoft Dynamics NAV\130\Service\` |
| NAV 2018 (v11) | `C:\Program Files (x86)\Microsoft Dynamics NAV\110\RoleTailored Client\finsql.exe` | `C:\Program Files\Microsoft Dynamics NAV\110\Service\` |
| NAV 2017 (v10) | `C:\Program Files (x86)\Microsoft Dynamics NAV\100\RoleTailored Client\finsql.exe` | `C:\Program Files\Microsoft Dynamics NAV\100\Service\` |
| NAV 2016 (v9) | `C:\Program Files (x86)\Microsoft Dynamics NAV\90\RoleTailored Client\finsql.exe` | `C:\Program Files\Microsoft Dynamics NAV\90\Service\` |

### Docker Container Pitfalls

1. **`docker exec` / `docker cp` werkt niet** — Hyper-V isolatie blokkeert dit. Gebruik altijd `Invoke-ScriptInBcContainer` en `Copy-FileToNavContainer`.

2. **CRONUS demo licentie blokkeert import** — Fout `[18023703] File, Import, Text`. Vervang `Cronus.flf` op disk en herstart de service.

3. **`Import-NAVServerLicense` werkt niet voor containers** — Schrijft naar DB maar service laadt nog steeds licentie van disk. Vervang het bestand direct.

4. **Omgevingsvariabelen leeg in container** — `$env:navservicename`, `$env:navdatabasename` etc. zijn leeg. Gebruik in plaats daarvan `. 'c:\run\ServiceSettings.ps1'` om `$ServerInstance`, `$NavServiceName` etc. te laden.

5. **finsql.exe hangt bij direct aanroep met `usesyntheticmessages=yes`** — Gebruik GEEN `usesyntheticmessages=yes` op NL containers. Roep finsql.exe aan zonder dit flag; als het hangt, check of de import al geslaagd was.

6. **`Import-NAVApplicationObject` met NavServer faalt** met `[18023703]` ook na licentie-fix — Dit is een timing issue. Na het vervangen van Cronus.flf de service herstarten en 10 seconden wachten voor het import-commando.

7. **`Import-NAVApplicationObject` meldt "OK" maar schrijft niets** — De navcommandresult.txt bevat dan `[0] The command completed successfully in '0' seconds` maar de SQL Object tabel is leeg. Oorzaak: finsql.exe connecteert via NavServer die de schema sync uitvoert maar de transaction rollback-t bij fout. Oplossing: gebruik directe finsql.exe aanroep ZONDER NavServer params.

8. **Licentielimiet: max 2 full users** — CRONUS demo licentie staat 2 gebruikers toe. ADMIN (default) + 1 extra. Voeg nooit `NT AUTHORITY\SYSTEM` toe als je ook `containeradministrator` nodig hebt.

9. **Object Type nummers** — In de SQL Object tabel: Type 0=Table, Type 5=Codeunit (niet Type 4 zoals je verwacht). Controleer altijd met `SELECT [Type], [Name] FROM [Object] WHERE [Name]='bekende naam'` om de lokale type-mapping te verifiëren.

10. **Container naam max 15 tekens** — BcContainerHelper geeft waarschuwing maar maakt de container wel aan. Houd namen kort.

---

## Regels

- Gebruik **altijd** NavServer params bij `Import-NAVApplicationObject` voor compile — zonder NavServer wordt Object Metadata niet geschreven
- In **Docker containers**: gebruik finsql.exe direct (ZONDER NavServer) voor import, dan `Compile-NAVApplicationObject` MET NavServer voor schema sync + Object Metadata
- **Nooit** `finsql.exe` met `usesyntheticmessages=yes` op Nederlandse BC — dit faalt met taalkwestie ("yes" niet geldig, verwacht "Ja/Nee")
- **Altijd** database naam ophalen via `Get-NAVServerConfiguration` of CustomSettings.config, nooit hardcoden
- Bij twijfel over SyncMode: gebruik `Force` in plaats van `Yes` — veiliger, maar check op dataverlies
- Compileer **altijd** via NavServer (Management port), niet direct SQL
- Test **altijd** via `Invoke-NAVCodeunit` of event log na import
- Sla de compiled versie op in git (na export uit DB), niet het handmatig bewerkte bestand
- Bij compilatieproblemen: zie `cal-compile-troubleshooting.md` — loop er niet blind doorheen
