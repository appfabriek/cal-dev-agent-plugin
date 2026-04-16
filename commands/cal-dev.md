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

## Regels

- Gebruik **altijd** NavServer params bij `Import-NAVApplicationObject` voor compile — zonder NavServer wordt Object Metadata niet geschreven
- **Nooit** `finsql.exe` met `usesyntheticmessages=yes` op Nederlandse BC — dit faalt met taalkwestie ("yes" niet geldig, verwacht "Ja/Nee")
- **Altijd** database naam ophalen via `Get-NAVServerConfiguration`, nooit hardcoden
- Bij twijfel over SyncMode: gebruik `Force` in plaats van `Yes` — veiliger, maar check op dataverlies
- Compileer **altijd** via NavServer (Management port 7345), niet direct SQL
- Test **altijd** via `Invoke-NAVCodeunit` of event log na import
- Sla de compiled versie op in git (na export uit DB), niet het handmatig bewerkte bestand
- Bij compilatieproblemen: zie `cal-compile-troubleshooting.md` — loop er niet blind doorheen
