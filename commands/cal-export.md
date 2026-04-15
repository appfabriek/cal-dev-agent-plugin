---
name: cal-export
description: Export C/AL objects from a NAV/BC database to text files via the runner
allowed-tools: Bash, Read, Write, Glob
---

# CAL Export

Exporteer C/AL objecten uit de database naar `.txt` bestanden via de self-hosted runner.

## Input

$ARGUMENTS — wat te exporteren:
- (leeg) — alle eigen objecten (ID 50000–99999)
- "COD50000" of "Codeunit 50000" — één specifiek object
- "50000..50099" — range
- "gewijzigd" — alleen objecten met Modified=Yes
- "alles" — alle objecten inclusief Microsoft base (waarschuwing: groot)
- "TAB" of "table" — alleen tabellen

## Instructies

### Stap 0 — Laad kennis

Lees `cal-tools.md` en `cal-git-patterns.md`:
```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-tools.md" 2>/dev/null | head -1
```

Lees ook CLAUDE.md voor instance-namen en database-configuratie.

### Stap 1 — Bepaal filter

| Input | Filter |
|-------|--------|
| leeg | `ID=50000..99999` |
| "COD50000" | `Type=Codeunit;ID=50000` |
| "50000..50099" | `ID=50000..50099` |
| "gewijzigd" | `Modified=Yes;ID=50000..99999` |
| "alles" | *(leeg — vraag bevestiging, kan 5000+ objecten zijn)* |
| "TAB" | `Type=Table;ID=50000..99999` |

### Stap 2 — Export via runner

```powershell
$ErrorActionPreference = 'Stop'

$cfg      = Get-NAVServerConfiguration -ServerInstance $env:BC_INSTANCE
$dbServer = $cfg.DatabaseServer
$dbName   = $cfg.DatabaseName
$outDir   = Join-Path $env:REPO_ROOT "objects"

New-Item -ItemType Directory -Path $outDir -Force | Out-Null

$filter  = "ID=50000..99999"  # pas aan op basis van input
$objects = Get-NAVApplicationObject `
    -DatabaseServer $dbServer -DatabaseName $dbName -Filter $filter

Write-Host "Exporteren: $($objects.Count) objecten naar $outDir"

$typeMap = @{
    Table='TAB'; Codeunit='COD'; Page='PAG'; Report='REP'
    XMLport='XML'; Query='QUE'; MenuSuite='MEN'; Form='FOR'; Dataport='DAT'
}

foreach ($obj in $objects) {
    $abbr     = $typeMap[$obj.Type.ToString()]
    $safeName = $obj.Name -replace '[\\/:*?"<>|]', '_'
    $file     = "$abbr$($obj.ID.ToString().PadLeft(5,'0')) - $safeName.txt"
    $outPath  = Join-Path $outDir $file

    Export-NAVApplicationObject `
        -DatabaseServer $dbServer -DatabaseName $dbName `
        -Path $outPath -Filter "Type=$($obj.Type);ID=$($obj.ID)"

    Write-Host "  $file"
}

Write-Host "=== Klaar: $($objects.Count) bestanden ===" -ForegroundColor Green
```

### Stap 3 — Datum/tijd ruis normaliseren

Na export, normaliseer datum/tijd om diff-ruis te vermijden:

```powershell
Get-ChildItem $outDir -Filter "*.txt" | ForEach-Object {
    $c = Get-Content $_.FullName -Raw
    $c = $c -replace '(?m)^\s+Date=\d{2}[-/]\d{2}[-/]\d{2,4};', '    Date=;'
    $c = $c -replace '(?m)^\s+Time=\d{2}:\d{2}:\d{2};', '    Time=;'
    Set-Content $_.FullName -Value $c -NoNewline
}
Write-Host "Datum/tijd genormaliseerd"
```

### Stap 4 — Rapporteer

Toon welke bestanden zijn aangemaakt/bijgewerkt, en suggereer de volgende stap:
- Nieuw: `git add objects/ && git status`
- Wijzigingen: `git diff objects/` om te reviewen vóór commit

## Regels

- Gebruik altijd NavAdminTool via runner — nooit finsql tenzij NavAdminTool ontbreekt (oude NAV versies)
- Exporteer altijd naar `.txt` (niet `.fob`) voor git-compatibiliteit
- Normaliseer datum/tijd altijd na export
- Bij "alles" (inclusief Microsoft base): vraag bevestiging — output kan honderden MB zijn
