# C/AL Tools — Export, Import en Compilatie

Referentie voor het werken met C/AL objecten via NavAdminTool, finsql.exe en de NAV Development Shell.

---

## Export-NAVApplicationObject

Exporteert C/AL objecten uit de database naar `.txt` of `.fob`.

```powershell
# Alle objecten exporteren naar tekstbestanden
Export-NAVApplicationObject `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -Path "C:\export\objects.txt" `
  -Filter "Type=Codeunit;ID=50000..50099"

# Enkel object
Export-NAVApplicationObject `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -Path "C:\export\COD50000.txt" `
  -Filter "Type=Codeunit;ID=50000"

# Als FOB (binair)
Export-NAVApplicationObject `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -Path "C:\export\objects.fob" `
  -Filter "Type=Table;ID=50000..50099" `
  -ExportToNewSyntax $false
```

### Filter syntax
```
"Type=Codeunit"                    → alle codeunits
"Type=Codeunit;ID=50000"           → één codeunit
"Type=Codeunit;ID=50000..50099"    → range
"ID=50000..50099"                  → alle types in range
"Modified=Yes"                     → alleen gewijzigde objecten
"Type=Table,Codeunit;ID=50000..99999" → tabellen EN codeunits
"VersionList=*MYAPP*"              → objecten met MYAPP in version list
```

### Tip: exporteer per object naar afzonderlijke bestanden
```powershell
$dbServer = "localhost"
$dbName   = "MyDatabase"
$outDir   = "C:\git\myproject\objects"

# Alle objecttypen exporteren als afzonderlijke bestanden
$types = @('Table','Codeunit','Page','Report','XMLport','Query')
foreach ($type in $types) {
    $objects = Get-NAVApplicationObject `
        -DatabaseServer $dbServer `
        -DatabaseName $dbName `
        -Filter "Type=$type;ID=50000..99999"

    foreach ($obj in $objects) {
        $safeName = $obj.Name -replace '[\\/:*?"<>|]', '_'
        $fileName = "$type.Substring(0,3).ToUpper()$($obj.ID.ToString().PadLeft(5,'0')) - $safeName.txt"
        Export-NAVApplicationObject `
            -DatabaseServer $dbServer `
            -DatabaseName $dbName `
            -Path (Join-Path $outDir $fileName) `
            -Filter "Type=$type;ID=$($obj.ID)"
    }
}
```

---

## Import-NAVApplicationObject

Importeert C/AL objecten in de database. **Dit compileert tegelijkertijd.**

```powershell
# Enkel bestand importeren
Import-NAVApplicationObject `
  -Path "C:\export\COD50000.txt" `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -ImportAction Overwrite `
  -SynchronizeSchemaChanges Force

# Met NAV service instance (voor schema sync via service)
Import-NAVApplicationObject `
  -Path "C:\export\COD50000.txt" `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -NavServerInstance $instance `
  -NavServerManagementPort 7045 `
  -ImportAction Overwrite `
  -SynchronizeSchemaChanges Force `
  -Confirm:$false
```

### ImportAction waarden
| Waarde | Gedrag |
|--------|--------|
| `Overwrite` | Overschrijft bestaand object — gebruik dit altijd |
| `Skip` | Slaat over als al bestaat |
| `Default` | Vraagt om bevestiging — vermijd in scripts |

### SynchronizeSchemaChanges waarden
| Waarde | Wanneer |
|--------|---------|
| `Yes` | Normale schema-wijzigingen (velden toevoegen) |
| `No` | Geen schema-wijziging (alleen code) |
| `Force` | Destructieve wijzigingen (velden verwijderen, type wijzigen) — **dataverlies mogelijk** |

---

## Get-NAVApplicationObject

Vraag objectmetadata op zonder te exporteren.

```powershell
# Alle eigen objecten
Get-NAVApplicationObject `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -Filter "ID=50000..99999" |
  Select-Object Type, ID, Name, Modified, VersionList |
  Sort-Object Type, ID |
  Format-Table -AutoSize

# Gewijzigde objecten
Get-NAVApplicationObject `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -Filter "Modified=Yes" |
  Select-Object Type, ID, Name, VersionList

# Objecten met specifieke version list tag
Get-NAVApplicationObject `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -Filter "VersionList=*MYAPP*"
```

---

## Get-NAVApplicationObjectProperty / Set-NAVApplicationObjectProperty

Lees of schrijf eigenschappen van objecten zonder het volledige object te exporteren.

```powershell
# Versie uitlezen
Get-NAVApplicationObjectProperty `
  -Source "C:\export\COD50000.txt" |
  Select-Object Type, Id, Name, VersionList, Modified, Date

# Version list updaten na wijziging
Set-NAVApplicationObjectProperty `
  -TargetPath "C:\export\COD50000.txt" `
  -VersionListProperty "NAVW19.00,MYAPP1.06" `
  -ModifiedProperty Yes `
  -DateTimeProperty (Get-Date)
```

---

## finsql.exe — Command Line Interface

finsql.exe is de command-line interface van de Classic Client / C/SIDE. Gebruik dit voor taken die niet via PowerShell kunnen, of voor oudere NAV-versies zonder NavAdminTool.

```powershell
$finsql = "C:\Program Files (x86)\Microsoft Dynamics NAV\110\RoleTailored Client\finsql.exe"
# of voor BC14:
$finsql = "C:\Program Files (x86)\Microsoft Dynamics 365 Business Central\140\RoleTailored Client\finsql.exe"

# Exporteren
& $finsql command=exportobjects, `
  file="C:\export\objects.txt", `
  database="$dbName", `
  servername="$dbServer", `
  filter="Type=Codeunit;ID=50000..50099", `
  logfile="C:\export\finsql.log"

# Importeren
& $finsql command=importobjects, `
  file="C:\export\COD50000.txt", `
  database="$dbName", `
  servername="$dbServer", `
  ImportAction=overwrite, `
  SynchronizeSchemaChanges=force, `
  logfile="C:\export\finsql.log"

# Compileren (apart van importeren)
& $finsql command=compileobjects, `
  database="$dbName", `
  servername="$dbServer", `
  filter="Type=Codeunit;ID=50000..50099", `
  logfile="C:\export\finsql.log"
```

### finsql logfile lezen
```powershell
# Controleer op fouten na finsql commando
$log = Get-Content "C:\export\finsql.log" -ErrorAction SilentlyContinue
if ($log) {
    Write-Host "finsql log:" -ForegroundColor Yellow
    $log | ForEach-Object { Write-Host "  $_" }
    # Fout als log niet leeg is (finsql schrijft alleen bij fouten)
    throw "finsql melding — zie log"
}
Write-Host "finsql: geen fouten" -ForegroundColor Green
```

---

## Compilatiestatus controleren

Na importeren: zijn objecten correct gecompileerd?

```powershell
# Objecten met compilatiefouten
Get-NAVApplicationObject `
  -DatabaseServer $dbServer `
  -DatabaseName $dbName `
  -Filter "Compiled=No;ID=50000..99999" |
  ForEach-Object {
    Write-Host "NIET GECOMPILEERD: $($_.Type) $($_.ID) - $($_.Name)" -ForegroundColor Red
  }
```

---

## Merge-NAVApplicationObject

Samenvoegen van twee versies van een object (bijv. bij upgrade).

```powershell
Merge-NAVApplicationObject `
  -OriginalPath "C:\base\COD50000.txt" `
  -ModifiedPath "C:\modified\COD50000.txt" `
  -TargetPath "C:\merged\COD50000.txt" `
  -ResultPath "C:\merged\result.txt"
```

`-ResultPath` bevat het merge-resultaat met conflicten gemarkeerd als `<<<< ORIGINAL` / `>>>> MODIFIED`. Conflicten moeten handmatig opgelost worden.

---

## Database naam ophalen via NavAdminTool

```powershell
# Instance-configuratie voor database connectie
$cfg    = Get-NAVServerConfiguration -ServerInstance $env:BC_INSTANCE
$dbServer = $cfg.DatabaseServer
$dbName   = $cfg.DatabaseName
```

Gebruik dit altijd i.p.v. hardcoded databasenamen.
