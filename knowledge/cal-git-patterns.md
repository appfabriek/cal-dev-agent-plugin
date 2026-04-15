# C/AL Git Patronen

Strategieën voor versiebeheer van C/AL objecten in git.

---

## Het probleem

C/AL broncode zit in de database — niet op disk. Git begrijpt geen databases. De oplossing is een **export-first workflow**: exporteer objecten naar tekstbestanden, commit die, en importeer voor deployment.

---

## Repository structuur

```
myproject/
├── objects/                    ← C/AL objecten als .txt bestanden
│   ├── TAB50000 - My Table.txt
│   ├── COD50000 - My Codeunit.txt
│   ├── PAG50000 - My Page.txt
│   └── ...
├── reports/                    ← RDLC layout bestanden (.rdlc)
│   └── REP50000 - My Report.rdlc
├── fob/                        ← Optioneel: binaire FOB backups
│   └── objects_20240101.fob   ← NIET committen tenzij noodzakelijk
├── scripts/                    ← Deployment scripts
│   ├── Export-Objects.ps1
│   ├── Import-Objects.ps1
│   └── Deploy.ps1
├── .gitignore
└── README.md
```

### .gitignore
```gitignore
# Binaire FOB bestanden (niet diffbaar)
*.fob

# finsql logbestanden
*.log
finsql.log

# Tijdelijke exportbestanden
export_temp/
*.bak
```

---

## Naamgevingsconventie voor objectbestanden

```
<TypeAfkorting><ID5cijfers> - <Naam>.txt
```

| Objecttype | Afkorting | Voorbeeld |
|------------|-----------|-----------|
| Table | TAB | `TAB50000 - Customer Extension.txt` |
| Codeunit | COD | `COD50000 - Sales Helper.txt` |
| Page | PAG | `PAG50000 - Customer Card Ext.txt` |
| Report | REP | `REP50000 - Customer List.txt` |
| XMLport | XML | `XML50000 - Import Orders.txt` |
| Query | QUE | `QUE50000 - Open Orders.txt` |
| MenuSuite | MEN | `MEN50000 - Main Menu.txt` |

ID altijd 5 cijfers met voorloopnullen zodat alfabetische sortering = numerieke sortering.

---

## Export script voor git

```powershell
# Export-Objects.ps1
# Exporteer alle eigen objecten naar afzonderlijke bestanden

param(
    [string]$Instance    = $env:BC_INSTANCE,
    [string]$OutputDir   = "$PSScriptRoot\..\objects",
    [string]$IdRange     = "50000..99999",
    [switch]$ModifiedOnly
)

$ErrorActionPreference = 'Stop'

$cfg      = Get-NAVServerConfiguration -ServerInstance $Instance
$dbServer = $cfg.DatabaseServer
$dbName   = $cfg.DatabaseName

New-Item -ItemType Directory -Path $OutputDir -Force | Out-Null

$filter = "ID=$IdRange"
if ($ModifiedOnly) { $filter += ";Modified=Yes" }

$objects = Get-NAVApplicationObject `
    -DatabaseServer $dbServer `
    -DatabaseName $dbName `
    -Filter $filter

Write-Host "Te exporteren: $($objects.Count) objecten" -ForegroundColor Cyan

$typeMap = @{
    Table     = 'TAB'; Codeunit  = 'COD'; Page      = 'PAG'
    Report    = 'REP'; XMLport   = 'XML'; Query     = 'QUE'
    MenuSuite = 'MEN'; Form      = 'FOR'; Dataport  = 'DAT'
}

foreach ($obj in $objects) {
    $abbr     = $typeMap[$obj.Type.ToString()]
    $safeName = $obj.Name -replace '[\\/:*?"<>|]', '_'
    $fileName = "$abbr$($obj.ID.ToString().PadLeft(5,'0')) - $safeName.txt"
    $outPath  = Join-Path $OutputDir $fileName

    Export-NAVApplicationObject `
        -DatabaseServer $dbServer `
        -DatabaseName $dbName `
        -Path $outPath `
        -Filter "Type=$($obj.Type);ID=$($obj.ID)"

    Write-Host "  $fileName"
}

Write-Host "Export klaar: $($objects.Count) bestanden in $OutputDir" -ForegroundColor Green
```

---

## Datum/tijd ruis in diffs verwijderen

Het exportformaat bevat datum/tijd in OBJECT-PROPERTIES. Dit veroorzaakt diff-ruis wanneer objecten op verschillende machines of momenten geëxporteerd worden.

```powershell
# Normaliseer datum/tijd in geëxporteerde .txt bestanden
# Vervang: Date=01-01-24;  →  Date=;
# Vervang: Time=12:00:00;  →  Time=;

Get-ChildItem $OutputDir -Filter "*.txt" | ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    $normalized = $content `
        -replace '(?m)^\s+Date=\d{2}-\d{2}-\d{2};', '    Date=;' `
        -replace '(?m)^\s+Time=\d{2}:\d{2}:\d{2};', '    Time=;'
    Set-Content $_.FullName -Value $normalized -NoNewline
}
```

Of gebruik een `.gitattributes` filter (geavanceerd):
```
objects/*.txt filter=cal-normalize
```

---

## Commit strategie

**Één object per commit** (voorkeur):
```bash
git add "objects/COD50000 - Sales Helper.txt"
git commit -m "fix(sales): herstel nul-deling in SalesHelper.GetDiscount (#42)"
```

**Gerelateerde objecten samen** (bij feature):
```bash
git add "objects/TAB50000 - Order Extension.txt"
git add "objects/PAG50001 - Order Extension.txt"
git add "objects/COD50002 - Order Extension Mgt.txt"
git commit -m "feat(order): voeg leveringstermijn toe aan orderextensie (#38)"
```

**Nooit**: bulk-commit van alle gewijzigde objecten zonder context.

---

## Diff lezen

C/AL tekst-exports zijn goed diffbaar. Let op deze patronen:

```diff
-  { 5   ;   ;OldField            ;Text50         }
+  { 5   ;   ;OldField            ;Text100        }
```
→ Veldlengte vergroot — schema-wijziging nodig bij import.

```diff
+  { 10  ;   ;NewField            ;Decimal        ;CaptionML=[ENU=New Field] }
```
→ Nieuw veld toegevoegd — `SynchronizeSchemaChanges=Yes` voldoende.

```diff
-  { 10  ;   ;OldField            ;Text50         }
```
→ Veld verwijderd — **dataverlies** bij `SynchronizeSchemaChanges=Force`.

---

## Branching

Zelfde model als AL (zie GITFLOW.md in het project), maar extra stap:

```
feature/42-description
    ↓ export objecten
    ↓ git commit
    ↓ PR / review
    ↓ merge naar main
    ↓ import naar omgevingen
```

**Bij merge conflicts in .txt bestanden:**
1. Begrijp het conflict (welk deel van het object is aangepast?)
2. Los op in de .txt editor of via `Merge-NAVApplicationObject`
3. Test het samengevoegde object door importeren en compileren
4. Commit de opgeloste versie

---

## Modified=Yes flag resetten

Na import zet NAV `Modified=Yes` op objecten. Reset dit bij release:

```powershell
Set-NAVApplicationObjectProperty `
  -TargetPath "objects\COD50000 - My Codeunit.txt" `
  -ModifiedProperty No
```

Of via finsql voor alle objecten in een database na deployment.
