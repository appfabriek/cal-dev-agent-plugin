# C/AL Compile Troubleshooting

Alle bekende valkuilen en oplossingen bij het compileren van C/AL objecten in BC14/NAV.
Lees dit bestand **vóór** je begint te debuggen — elke fout hier is al eens de lange weg doorlopen.

---

## Overzicht diagnose-stroom

```
Import geslaagd?
  └─ Nee → Zie "Import-fout"
  └─ Ja, maar Compiled=leeg → Zie "Object niet gecompileerd na import"
      └─ Developer Services 503? → Zie "Developer Services 503"
      └─ Tenant Failed? → Zie "Tenant Failed / NavTenantNotAccessibleException"
          └─ Versie-mismatch? → Zie "Versie-mismatch $ndo$tenantproperty"
      └─ Compile returned empty maar geen fout?
          └─ Zonder NavServer? → Object Metadata NIET geschreven → Zie "Compile zonder NavServer"
Invoke-NAVCodeunit mislukt?
  └─ "Access is denied to company X" → Zie "Permissie-fout Invoke-NAVCodeunit"
  └─ "Het systeem is niet toegankelijk, Failed" → Tenant Failed, zie boven
  └─ "Metadata is out of sync" → Zie "Object Metadata out of sync"
```

---

## 1. Compile zonder NavServer — Object Metadata wordt NIET geschreven

**Symptoom:** `Import-NAVApplicationObject` of `Compile-NAVApplicationObject` zonder `-NavServerName`/`-NavServerInstance`/`-NavServerManagementPort` parameters loopt zonder fout, maar daarna:
- `Compiled=` (leeg of False) in het geëxporteerde object
- Tenant geeft `NavMetadataNotFoundException` of `Metadata is out of sync` voor het object

**Oorzaak:** Zonder NavServer params werkt Import-NAVApplicationObject als directe SQL-operatie. Het schrijft de C/AL broncode naar de `Object` tabel maar **schrijft geen Object Metadata** (de gecompileerde .NET assembly + XML schema). De BC service kan het object dan niet uitvoeren.

**Oplossing:** Gebruik altijd NavServer params:
```powershell
Import-NAVApplicationObject `
    -Path $file `
    -DatabaseServer $dbServer -DatabaseName $dbName `
    -NavServerName 'localhost' `
    -NavServerInstance $inst `
    -NavServerManagementPort 7345 `
    -ImportAction Overwrite `
    -SynchronizeSchemaChanges Force `
    -Confirm:$false
```

**Let op:** Als de service niet bereikbaar is (tenant Failed, 503), zal ook de NavServer-variant mislukken. Los de service-problemen eerst op.

---

## 2. `Compile-NAVApplicationObject -SynchronizeSchemaChanges No` slaat Modified=True over

**Symptoom:** Compile geeft geen fout, maar het object is alsnog niet gecompileerd.

**Oorzaak:** `SynchronizeSchemaChanges=No` slaat objecten met `Modified=True` automatisch over. Een object staat op `Modified=True` wanneer het recent geïmporteerd is zonder schema-sync te voltooien.

**Oplossing A:** Gebruik `SynchronizeSchemaChanges=Force` met NavServer params (aanbevolen).

**Oplossing B (noodoplossing, alleen als de SQL schema al correct is):**
1. Exporteer het object
2. Verander `Modified=Yes;` naar `Modified=No;` in het OBJECT-PROPERTIES blok
3. Importeer terug (direct SQL, geen NavServer)
4. Compileer met `SynchronizeSchemaChanges=No`

> ⚠️ Pas op: stap B werkt alleen als de SQL tabel al alle verwachte kolommen heeft. Als het schema nog niet klopt, zal het object compileren maar de tenant crashen met "Metadata is out of sync".

---

## 3. `finsql.exe` met `usesyntheticmessages=yes` — taalfout op Nederlandse BC

**Symptoom:** finsql scherm of log toont:
```
Optie 'yes' is niet toegestaan. Geldige keuzemogelijkheden zijn: Nee, Ja
```

**Oorzaak:** `usesyntheticmessages=yes` geeft het Engelse woord "yes" als antwoord op BC-dialogen. Op een Nederlandse BC-installatie verwachten dialogen "Ja" of "Nee", niet "yes".

**Oplossing:** Gebruik `usesyntheticmessages=yes` NIET. Gebruik in plaats daarvan `Import-NAVApplicationObject` of `Compile-NAVApplicationObject` via PowerShell — die geven geen interactieve dialogen.

Als je toch finsql moet gebruiken (oudere NAV), gebruik dan een auto-click script dat specifiek zoekt naar de knop met tekst "Ja":
```powershell
$jaBtn = [WinAPI2]::FindWindowEx($hwnd, [IntPtr]::Zero, "Button", "Ja")
```

---

## 4. Developer Services 503

**Symptoom:** `Invoke-WebRequest "http://localhost:9049/$inst/"` geeft HTTP 503.

**Gevolg:** `Compile-NAVApplicationObject -SynchronizeSchemaChanges Force` met NavServer params mislukt met:
```
[27590710] Kan geen tabelwijzigingen verwerken omdat de Microsoft Dynamics NAV Development Environment geen verbinding kan maken
```

**Diagnose stap 1 — Zijn Developer Services ingeschakeld?**
```powershell
# Check config bestand direct (betrouwbaarder dan Get-NAVServerConfiguration filtering)
$cfg = 'C:\Program Files\Microsoft Dynamics 365 Business Central\140\Service\Instances\BC140-KMT\CustomSettings.config'
$xml = [xml](Get-Content $cfg)
$xml.SelectNodes('//add') | Where-Object { $_.key -match 'Developer' } |
    ForEach-Object { Write-Host "$($_.key) = $($_.value)" }
```

**Diagnose stap 2 — Is de tenant Operational?**
```powershell
(Get-NAVTenant -ServerInstance $inst -Tenant default).State
# Moet 'Operational' zijn, niet 'Failed'
```

Als tenant Failed is, zal Developer Services altijd 503 blijven, ook als de instelling op `true` staat. Los de tenant-fout eerst op.

**Oplossing als instelling false is:**
```powershell
Set-NAVServerConfiguration -ServerInstance $inst -KeyName 'DeveloperServicesEnabled' -KeyValue 'true'
Restart-NAVServerInstance -ServerInstance $inst
# Wacht tot service Running is, daarna re-check
```

**Oplossing als instelling true is maar toch 503:**
De tenant is niet Operational. Ga naar sectie 5 of 6.

---

## 5. Tenant Failed — NavTenantNotAccessibleException

**Symptoom:** 
- `(Get-NAVTenant -ServerInstance $inst -Tenant default).State` = `Failed`
- Event log: `NavTenantNotAccessibleException — De tenant default is niet toegankelijk`
- `Invoke-NAVCodeunit` faalt met "Het systeem is niet toegankelijk, omdat het de status Failed heeft"

**Diagnose:**
```powershell
$t = Get-NAVTenant -ServerInstance $inst -Tenant default
Write-Host $t.State
Write-Host $t.DetailedState
```

Twee hoofdoorzaken:

### 5a. Versie-mismatch (meest voorkomend)
`DetailedState` toont:
```
Tenant default cannot be used because it is assigned a data version (14.53.50097.0)
that is greater than the Application Version (14.0.48501.0) supported on the server instance.
```
→ Zie sectie 6.

### 5b. Object Metadata mist / out of sync
`DetailedState` is leeg, maar event log toont:
```
Message Metadata is out of sync -- Table:50000
at NavSqlDatabaseSync.CheckTableInSync(...)
```
→ Compile het betreffende tabel-object opnieuw via Import met NavServer + SynchronizeSchemaChanges=Force.
→ Als dat mislukt (Dev Services 503), los eerst de versie-mismatch op (sectie 6).

---

## 6. Versie-mismatch — $ndo$tenantproperty.applicationversion

**Symptoom:**
```
Tenant cannot be used because it is assigned a data version (X) that is greater than
the Application Version (Y) supported on the Microsoft Dynamics 365 Business Central Server instance.
```

**Oorzaak:** De tabel `$ndo$tenantproperty` in de database bevat een `applicationversion` die hoger is dan de versie van de BC service.

**Wanneer dit optreedt:**
- Na `Sync-NAVTenant -Mode ForceSync` waarbij de objecten metadata-versies hadden van een nieuwere BC-build
- Na gebruik van een nieuwere BC service en daarna terugschakelen naar een oudere
- Bij import van objecten vanuit een nieuwere BC-omgeving

**Diagnose:**
```powershell
# Service versie
(Get-NAVServerInstance -ServerInstance $inst).Version

# Tenant data versie
(Get-NAVTenant -ServerInstance $inst -Tenant default -ErrorAction SilentlyContinue).DetailedState

# Of direct uit de database
$tbl = '$ndo$tenantproperty'
Invoke-Sqlcmd -ServerInstance $dbServer -Database $dbName `
    -Query "SELECT tenantid, applicationversion FROM [dbo].[$tbl]"
```

**Oplossing — reset de versie naar de service versie:**
```powershell
$serviceVersion = (Get-NAVServerInstance -ServerInstance $inst).Version
$tbl = '$ndo$tenantproperty'
Invoke-Sqlcmd -ServerInstance $dbServer -Database $dbName `
    -Query "UPDATE [dbo].[$tbl] SET applicationversion = '$serviceVersion' WHERE tenantid = 'default'"
Write-Host "Versie gereset naar $serviceVersion"
```

Herstart daarna de service en run een ForceSync:
```powershell
Restart-NAVServerInstance -ServerInstance $inst
# Wacht op Running...
Sync-NAVTenant -ServerInstance $inst -Mode ForceSync -Force
(Get-NAVTenant -ServerInstance $inst -Tenant default).State
# Moet 'Operational' of 'OperationalWithSyncPending' zijn
```

> ⚠️ **Voorzichtigheid met Sync-NAVTenant -Mode ForceSync:** Dit commando kan de `applicationversion` opnieuw updaten als er objecten in de database zijn met metadata van een nieuwere BC-versie. Gebruik het alleen als de tenant Operational is en Developer Services niet 503 geven.

---

## 7. Object Metadata out of sync — "Metadata is out of sync -- Table:X"

**Symptoom:** Event log toont:
```
Message Metadata is out of sync -- Table:50000
at Microsoft.Dynamics.Nav.Runtime.NavSqlDatabaseSync.CheckTableInSync(Int32 tableThatMustBeInSync)
```

**Oorzaak:** De `Object Metadata.Metadata` XML voor tabel X beschrijft een schema dat niet overeenkomt met de SQL tabel. Dit treedt op wanneer:
- Een veld is toegevoegd aan de SQL tabel maar Object Metadata is niet geüpdatet (object niet gecompileerd)
- Object Metadata is leeg/corrupt

**Oplossing:**
1. Exporteer het object: `Export-NAVApplicationObject -Filter "Type=Table;ID=X"`
2. Controleer of het veld aanwezig is in de C/AL broncode
3. Importeer opnieuw met NavServer params + `SynchronizeSchemaChanges=Force`
4. Controleer: `Compiled=Yes` in export na import

---

## 8. Permissie-fout Invoke-NAVCodeunit

**Symptoom:** `Invoke-NAVCodeunit` geeft:
```
Access is denied to company 'KMT Test'
```

**Oorzaak:** De Windows-gebruiker die het PowerShell script uitvoert heeft geen `SUPER` (of voldoende) permissies in het BC-bedrijf.

**Diagnose:**
```powershell
# Huidige gebruiker
Write-Host "$env:USERDOMAIN\$env:USERNAME"

# Configureerde BC gebruikers
Get-NAVServerUser -ServerInstance $inst | Select-Object UserName, State | Format-Table
```

**Oplossing:**
```powershell
$company = 'KMT Test'  # pas aan
New-NAVServerUserPermissionSet `
    -ServerInstance $inst `
    -WindowsAccount "$env:USERDOMAIN\$env:USERNAME" `
    -PermissionSetId SUPER `
    -CompanyName $company
```

Daarna opnieuw uitvoeren — geen herstart nodig.

---

## 9. Verkeerde database naam

**Symptoom:** Import of export lijkt te werken maar objecten staan niet in de database, of finsql zegt "database not found".

**Oorzaak:** Database naam hardcoded als `KMT`, `CRONUS`, etc. terwijl de echte naam bijv. `kmt-test` (met koppelteken) is.

**Oplossing — altijd naam ophalen via config:**
```powershell
$dbServer = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DatabaseServer').Value
$dbName   = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DatabaseName').Value
Write-Host "Database: $dbServer\$dbName"
```

---

## 10. Verkeerde NavAdminTool geladen (BC27 i.p.v. BC14)

**Symptoom:** NavAdminTool cmdlets geven vreemde fouten of zijn niet gevonden, of de versie van instance klopt niet.

**Oorzaak:** Op een machine met meerdere BC-versies (BC14, BC25, BC27) wordt de verkeerde NavAdminTool geladen.

**Oplossing — laad altijd het expliciete versiepad:**
```powershell
# BC14
Import-Module 'C:\Program Files\Microsoft Dynamics 365 Business Central\140\Service\NavAdminTool.ps1' -WarningAction SilentlyContinue

# Model Tools (voor Export/Import/Compile):
Import-Module 'C:\Program Files (x86)\Microsoft Dynamics 365 Business Central\140\RoleTailored Client\Microsoft.Dynamics.Nav.Model.Tools.psd1' -WarningAction SilentlyContinue
```

Controleer welke instances beschikbaar zijn op de machine:
```powershell
Get-NAVServerInstance | Select-Object ServerInstance, State, Version | Format-Table -AutoSize
```

---

## 11. Bedrijfsnaam ophalen voor Invoke-NAVCodeunit

Haal de bedrijfsnaam op via de NAS-configuratie (betrouwbaarder dan hardcoderen):
```powershell
$nasArg = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'NASServicesStartupArgument').Value
$company = if ($nasArg -match 'company=(.+)') { $Matches[1] } else { 'CRONUS' }
Write-Host "Bedrijf: $company"
```

---

---

## 12. Docker container — CRONUS licentie blokkeert import

**Symptoom:** `Import-NAVApplicationObject` of `Import-ObjectsToNavContainer` geeft:
```
[18023703] You do not have permission to run the 'File, Import, Text' System.
```
of `navcommandresult.txt` bevat fout maar PowerShell cmdlet rapporteert toch "OK" (non-terminating error) — objecten staan NIET in SQL.

**Oorzaak:** BcContainerHelper containers gebruiken de CRONUS demo licentie (`Cronus.flf`, ~7 KB). Deze licentie mist de "File, Import, Text" systeem-permissie die nodig is voor object import. `Import-NAVServerLicense` schrijft wel naar de DB maar de service laadt nog steeds de .flf van disk.

**Oplossing:**
```powershell
$licBytes = [System.IO.File]::ReadAllBytes('D:\repos\plants\license\BC14_DEV_NL.flf')

Invoke-ScriptInBcContainer -containerName $containerName -scriptblock {
    param($bytes)
    . 'c:\run\ServiceSettings.ps1'
    $cronusFlf = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV' -Recurse -Filter 'Cronus.flf' -ErrorAction SilentlyContinue |
        Select-Object -First 1 -ExpandProperty FullName
    [System.IO.File]::WriteAllBytes($cronusFlf, $bytes)
    Write-Host "Licentie vervangen: $cronusFlf"
    Restart-Service "MicrosoftDynamicsNavServer`$$ServerInstance" -Force
    Start-Sleep -Seconds 10
} -argumentList (,$licBytes)
```

**Verificatie:** Een correct werkende import logt `navcommandresult.txt` met: `[0] The command completed successfully in '0' seconds` EN de objecten zijn direct zichtbaar in SQL.

---

## 13. Docker container — `Import-NAVApplicationObject` meldt "OK" maar schrijft niets

**Symptoom:** Import zegt "GESLAAGD" of geen fout, maar SQL `SELECT FROM [Object] WHERE [ID]=50099` geeft niets.

**Oorzaak:** `Import-NAVApplicationObject` met NavServer params roept finsql.exe aan met `SynchronizeSchemaChanges=Force`. Als de NavServer de schema sync uitvoert maar een probleem tegenkomt, rollback-t finsql.exe de hele transactie zonder foutmelding aan de cmdlet terug te geven.

**Oplossing — gebruik twee stappen:**
1. Import via finsql.exe ZONDER NavServer params (directe SQL write, geen schema sync)
2. Compileer via `Compile-NAVApplicationObject` MET NavServer (schema sync + Object Metadata)

```powershell
Invoke-ScriptInBcContainer -containerName $containerName -scriptblock {
    param($objectStr)
    . 'c:\run\ServiceSettings.ps1'
    $finsql = Get-ChildItem 'C:\Program Files (x86)\Microsoft Dynamics NAV' -Recurse -Filter 'finsql.exe' -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
    $cfg    = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV' -Recurse -Filter 'CustomSettings.config' -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
    $raw    = Get-Content $cfg -Raw
    $db     = if ($raw -match 'key="DatabaseName"\s+value="([^"]*)"') { $Matches[1] } else { 'CRONUS' }
    $port   = if ($raw -match 'key="ManagementServicesPort"\s+value="([^"]*)"') { [int]$Matches[1] } else { 7045 }

    $enc  = [System.Text.Encoding]::GetEncoding(1252)
    $path = 'C:\temp\import.txt'
    [System.IO.File]::WriteAllText($path, $objectStr, $enc)

    # Stap 1: import via finsql.exe ZONDER NavServer
    $logPath = 'C:\temp\import.log'
    Start-Process -FilePath $finsql `
        -ArgumentList "command=importobjects,file=`"$path`",logfile=`"$logPath`",servername=`"localhost\SQLEXPRESS`",database=`"$db`",ntauthentication=yes" `
        -Wait -PassThru -NoNewWindow | Out-Null
    if ((Test-Path $logPath) -and (Get-Item $logPath).Length -gt 0) {
        Write-Error "finsql import fout: $(Get-Content $logPath)"
    }

    # Stap 2: compile MET NavServer
    $mgmtDll = Get-ChildItem 'C:\Program Files\Microsoft Dynamics NAV' -Recurse -Filter 'Microsoft.Dynamics.Nav.Management.dll' -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
    Import-Module $mgmtDll -ErrorAction SilentlyContinue -WarningAction SilentlyContinue
    Compile-NAVApplicationObject -DatabaseServer 'localhost\SQLEXPRESS' -DatabaseName $db `
        -Filter 'Type=Table;ID=50099' `
        -NavServerName 'localhost' -NavServerInstance $ServerInstance -NavServerManagementPort $port `
        -SynchronizeSchemaChanges Force -Recompile
} -argumentList $objectStr
```

---

## Snel diagnose script

Plak dit als je niet weet wat er mis is:

```powershell
$navAdmin   = 'C:\Program Files\Microsoft Dynamics 365 Business Central\140\Service\NavAdminTool.ps1'
$modelTools = 'C:\Program Files (x86)\Microsoft Dynamics 365 Business Central\140\RoleTailored Client\Microsoft.Dynamics.Nav.Model.Tools.psd1'
Import-Module $navAdmin   -WarningAction SilentlyContinue
Import-Module $modelTools -WarningAction SilentlyContinue

$inst = 'BC140-KMT'

Write-Host "=== Service ===" 
$svc = Get-NAVServerInstance -ServerInstance $inst
Write-Host "State: $($svc.State)  Versie: $($svc.Version)"

Write-Host "`n=== Tenant ==="
$t = Get-NAVTenant -ServerInstance $inst -Tenant default -ErrorAction SilentlyContinue
if ($t) {
    Write-Host "State: $($t.State)"
    if ($t.DetailedState) { Write-Host "DetailedState: $($t.DetailedState)" }
} else {
    Write-Host "(geen tenant info — single-tenant of tenant niet gemount)"
}

Write-Host "`n=== Database ==="
$dbServer = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DatabaseServer').Value
$dbName   = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DatabaseName').Value
Write-Host "$dbServer \ $dbName"

Write-Host "`n=== Developer Services ==="
$devEnabled = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DeveloperServicesEnabled').Value
$devPort    = (Get-NAVServerConfiguration -ServerInstance $inst | Where-Object Key -eq 'DeveloperServicesPort').Value
Write-Host "Enabled: $devEnabled  Port: $devPort"
try {
    Invoke-WebRequest "http://localhost:$devPort/$inst/" -TimeoutSec 4 -UseBasicParsing | Out-Null
    Write-Host "HTTP: OK"
} catch {
    Write-Host "HTTP: $($_.Exception.Message)"
}

Write-Host "`n=== Event log (laatste fouten) ==="
Get-EventLog -LogName Application -Source '*Dynamics*' -Newest 3 -EntryType Error -ErrorAction SilentlyContinue |
    ForEach-Object { Write-Host "[$($_.TimeGenerated)] $($_.Message.Substring(0,[Math]::Min(200,$_.Message.Length)))"; Write-Host "---" }
```
