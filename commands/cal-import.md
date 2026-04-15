---
name: cal-import
description: Import and compile C/AL objects into a NAV/BC database via the runner
allowed-tools: Bash, Read, Write, Glob
---

# CAL Import

Importeer en compileer C/AL objecten in de database via de self-hosted runner.

## Input

$ARGUMENTS — wat te importeren:
- (leeg) — alle gewijzigde `.txt` bestanden t.o.v. git HEAD
- "COD50000" of een bestandsnaam — één specifiek bestand
- "objects/" — alle bestanden in de objects/ map
- "force" — gebruik SynchronizeSchemaChanges=Force (destructief, vraagt bevestiging)

## Instructies

### Stap 0 — Laad kennis

Lees `cal-tools.md`:
```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-tools.md" 2>/dev/null | head -1
```

### Stap 1 — Bepaal bestanden en SyncMode

**Gewijzigde bestanden detecteren:**
```bash
git diff --name-only HEAD | grep "^objects/.*\.txt$"
git status --porcelain | grep "objects/.*\.txt"
```

**SyncMode bepalen op basis van diff:**
```bash
# Controleer op destructieve wijzigingen (velden verwijderen)
git diff HEAD objects/ | grep "^-  { [0-9]"
```

| Situatie | SyncMode |
|----------|----------|
| Alleen code-wijzigingen | `No` |
| Nieuwe velden toegevoegd | `Yes` |
| Velden verwijderd/hernoemd of type gewijzigd | `Force` — vraag **bevestiging** |
| Onbekend / nieuw object | `Yes` |

### Stap 2 — Import via runner

```powershell
$ErrorActionPreference = 'Stop'

$cfg      = Get-NAVServerConfiguration -ServerInstance $env:BC_INSTANCE
$dbServer = $cfg.DatabaseServer
$dbName   = $cfg.DatabaseName
$syncMode = "Yes"  # pas aan op basis van stap 1

# Lijst van te importeren bestanden
$files = @(
    # Vul in op basis van input
)

Write-Host "=== Import naar $env:BC_INSTANCE ($dbName) ===" -ForegroundColor Cyan
Write-Host "SynchronizeSchemaChanges: $syncMode"
Write-Host "Bestanden: $($files.Count)"
Write-Host ""

foreach ($file in $files) {
    Write-Host "Importeren: $(Split-Path $file -Leaf)..." -NoNewline

    $logFile = Join-Path $env:TEMP "cal-import-$(Get-Random).log"

    Import-NAVApplicationObject `
        -Path $file `
        -DatabaseServer $dbServer `
        -DatabaseName $dbName `
        -NavServerInstance $env:BC_INSTANCE `
        -NavServerManagementPort 7045 `
        -ImportAction Overwrite `
        -SynchronizeSchemaChanges $syncMode `
        -Confirm:$false

    # Controleer finsql log op fouten
    if ((Test-Path $logFile) -and (Get-Content $logFile)) {
        Write-Host " FOUT" -ForegroundColor Red
        Get-Content $logFile | ForEach-Object { Write-Host "  $_" }
    } else {
        Write-Host " OK" -ForegroundColor Green
    }
    Remove-Item $logFile -Force -ErrorAction SilentlyContinue
}

# Controleer compilatiestatus
Write-Host ""
Write-Host "=== Compilatiestatus ===" -ForegroundColor Cyan
$notCompiled = Get-NAVApplicationObject `
    -DatabaseServer $dbServer `
    -DatabaseName $dbName `
    -Filter "Compiled=No;ID=50000..99999"

if ($notCompiled.Count -gt 0) {
    Write-Host "NIET GECOMPILEERD:" -ForegroundColor Red
    $notCompiled | ForEach-Object { Write-Host "  $($_.Type) $($_.ID) - $($_.Name)" }
} else {
    Write-Host "Alle objecten correct gecompileerd." -ForegroundColor Green
}
```

### Stap 3 — Verifieer

Na import: controleer of de objecten correct werken via een snelle smoke test:
- Pagina openen (via `/diagnose` of handmatig in de client)
- Controleer event log op runtime-fouten
- Bij reports: controleer rendering

### Stap 4 — Rapporteer

Toon:
- Welke objecten zijn geïmporteerd
- Compilatiestatus
- Eventuele fouten met de volledige foutmelding

## Regels

- **Vraag altijd bevestiging bij `SynchronizeSchemaChanges=Force`** — dit kan data vernietigen
- Importeer in dependency-volgorde: tabellen → codeunits → pagina's → rapporten
- Controleer compilatiestatus altijd na import
- Gebruik NavAdminTool — val terug op finsql.exe alleen voor oudere NAV-versies zonder NavAdminTool
- Bij compilatiefouten: exporteer het object opnieuw, vergelijk met wat je wilde importeren
