---
name: cal-deploy
description: Deploy C/AL changes to one or more NAV/BC environments via the runner
allowed-tools: Bash, Read, Write, Glob
---

# CAL Deploy

Deployeer C/AL wijzigingen naar een of meerdere omgevingen via de self-hosted runner.

## Input

$ARGUMENTS — optionele instructies:
- (leeg) — deploy gewijzigde objecten (git diff vs vorige commit) naar dev
- "pop" of een instance-naam — specifieke omgeving
- "alle" — alle omgevingen sequentieel
- "force" — SynchronizeSchemaChanges=Force (vraagt bevestiging)
- "COD50000" of bestandsnaam — één specifiek object

## Instructies

### Stap 0 — Laad kennis

```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-tools.md" 2>/dev/null | head -1
```

Lees CLAUDE.md voor instance-namen en ForceSync-beleid per omgeving.

### Stap 1 — Bepaal te deployen objecten

```bash
# Gewijzigd t.o.v. vorige commit
git diff HEAD~1 HEAD --name-only objects/

# Of staged wijzigingen
git diff --cached --name-only objects/
```

Detecteer automatisch benodigde SyncMode door de diff te analyseren:
- Veld verwijderd/hernoemd → `Force` → vraag bevestiging
- Veld toegevoegd → `Yes`
- Alleen code → `No`

### Stap 2 — Deploy via runner

```powershell
$ErrorActionPreference = 'Stop'

$cfg      = Get-NAVServerConfiguration -ServerInstance $env:BC_INSTANCE
$dbServer = $cfg.DatabaseServer
$dbName   = $cfg.DatabaseName
$syncMode = "Yes"  # pas aan op basis van stap 1

# Bestanden te deployen (vanuit git diff)
$files = @(
    # Vul in op basis van stap 1
)

# Sorteer op dependency-volgorde: TAB → COD → PAG/REP/XML
$order = @{ TAB=1; COD=2; XML=3; PAG=4; REP=5; QUE=6; MEN=7 }
$files = $files | Sort-Object {
    $name = Split-Path $_ -Leaf
    if ($name -match '^([A-Z]{3})') { $order[$Matches[1]] ?? 99 } else { 99 }
}

Write-Host "=== Deploy naar $env:BC_INSTANCE ===" -ForegroundColor Cyan
Write-Host "SynchronizeSchemaChanges: $syncMode"
Write-Host ""

$success = 0
$failed  = 0

foreach ($file in $files) {
    $name = Split-Path $file -Leaf
    Write-Host "  $name..." -NoNewline

    try {
        Import-NAVApplicationObject `
            -Path (Join-Path $env:REPO_ROOT $file) `
            -DatabaseServer $dbServer `
            -DatabaseName $dbName `
            -NavServerInstance $env:BC_INSTANCE `
            -NavServerManagementPort 7045 `
            -ImportAction Overwrite `
            -SynchronizeSchemaChanges $syncMode `
            -Confirm:$false
        Write-Host " OK" -ForegroundColor Green
        $success++
    } catch {
        Write-Host " FOUT: $_" -ForegroundColor Red
        $failed++
    }
}

Write-Host ""
Write-Host "=== Resultaat: $success OK, $failed mislukt ===" -ForegroundColor $(if ($failed -gt 0) { 'Red' } else { 'Green' })

# Compilatiestatus
$notCompiled = Get-NAVApplicationObject `
    -DatabaseServer $dbServer -DatabaseName $dbName `
    -Filter "Compiled=No;ID=50000..99999"
if ($notCompiled.Count -gt 0) {
    Write-Host "NIET GECOMPILEERD:" -ForegroundColor Red
    $notCompiled | ForEach-Object { Write-Host "  $($_.Type) $($_.ID) - $($_.Name)" }
    exit 1
}
```

### Stap 3 — Verifieer deployment

Na deployen: controleer de omgeving:
```powershell
# Versie van gedeployede objecten
Get-NAVApplicationObject `
    -DatabaseServer $dbServer `
    -DatabaseName $dbName `
    -Filter "ID=50000..99999;Modified=Yes" |
    Select-Object Type, ID, Name, Date, VersionList |
    Sort-Object Type, ID
```

### Stap 4 — Meerdere omgevingen

Bij "alle": trigger **sequentieel** per instance via `bc-runner.yaml`. Wacht op resultaat vóór de volgende omgeving.

Rapporteer per omgeving: geslaagd / mislukt / overgeslagen.

## Regels

- **Deployment volgorde**: TAB → COD → XML → PAG → REP — tabellen altijd eerst
- **Vraag bevestiging** voor Force-sync en voor productie-omgevingen
- **Nooit parallel**: één instance per keer, zeker bij schema-wijzigingen
- Documenteer de benodigde SyncMode in de commit message zodat andere developers weten wat nodig is bij deployment
- Na deployment: check event log op runtime-fouten (`/bc-log` uit bc-dev-agent-plugin)
