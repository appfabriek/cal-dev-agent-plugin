---
name: cal-git
description: Git management for C/AL code - export strategy, meaningful diffs, merge conflict resolution
allowed-tools: Bash, Read, Write, Glob
---

# CAL Git

Git-beheer voor C/AL objecten: exporteren voor commit, diffs interpreteren en merge-conflicten oplossen.

## Input

$ARGUMENTS — optionele instructies:
- (leeg) of "status" — exporteer gewijzigde objecten en toon git status
- "diff" — toon zinvolle diff van gewijzigde objecten
- "commit [bericht]" — exporteer, normaliseer en commit gewijzigde objecten
- "merge [branch]" — help bij het oplossen van merge-conflicten in C/AL bestanden
- "setup" — zet de git-structuur op voor een nieuw C/AL project

## Instructies

### Stap 0 — Laad kennis

```bash
find ~/.claude/plugins/cal-dev-agent-plugin/knowledge \
     ~/code/cal-dev-agent-plugin/knowledge \
     -name "cal-git-patterns.md" 2>/dev/null | head -1
```

### Bij "status" of leeg

1. Vraag de database welke objecten zijn gewijzigd:
   ```powershell
   # Via runner: Modified=Yes objecten
   Get-NAVApplicationObject -DatabaseServer $dbServer -DatabaseName $dbName `
       -Filter "Modified=Yes;ID=50000..99999" |
       Select-Object Type, ID, Name, VersionList
   ```

2. Vergelijk met git:
   ```bash
   git status objects/
   git diff --stat objects/
   ```

3. Toon discrepantie: objecten in DB gewijzigd maar niet in git, of omgekeerd.

### Bij "diff"

Exporteer gewijzigde objecten tijdelijk en toon zinvolle diff:

```bash
# Gewijzigde bestanden
git diff HEAD objects/

# Interpreteer de diff:
# + regels = toegevoegd, - regels = verwijderd
# Let op: veldwijzigingen, nieuwe procedures, gewijzigde triggers
```

Samenvatting geven van wat er inhoudelijk is gewijzigd (niet alleen de regels).

### Bij "commit [bericht]"

1. Exporteer gewijzigde objecten via runner (`/cal-export gewijzigd`)
2. Normaliseer datum/tijd
3. Controleer diff op schema-wijzigingen (documenteer in commit message)
4. Commit:

```bash
# Commit per object (voorkeur)
git add "objects/COD50000 - My Codeunit.txt"
git commit -m "fix(codeunit): herstel berekening in MyProcedure (#42)"

# Of alle gewijzigde objecten samen
git add objects/
git commit -m "feat(order): voeg leveringstermijn toe (#38)

Gewijzigd:
- TAB50000: nieuw veld DeliveryDays (Integer)
- PAG50000: DeliveryDays toegevoegd aan General tab
- COD50000: berekening DeliveryDays in OnValidate

Schema: SynchronizeSchemaChanges=Yes vereist bij deployment"
```

### Bij "merge [branch]"

C/AL merge-conflicten in `.txt` bestanden oplossen:

```bash
# Toon bestanden met conflicten
git status | grep "both modified"

# Bekijk conflict in een bestand
git diff objects/COD50000\ -\ My\ Codeunit.txt
```

**Conflicten interpreteren:**

```
<<<<<<< HEAD
    PROCEDURE OldProcedure@1();
    BEGIN
      // onze versie
    END;
=======
    PROCEDURE OldProcedure@1();
    BEGIN
      // hun versie
    END;
>>>>>>> feature/other
```

Aanpak:
1. Begrijp beide versies — welke intentie had elk?
2. Combineer indien mogelijk (niet simpelweg één versie kiezen)
3. Importeer het samengevoegde object en compileer
4. Test
5. Markeer als opgelost: `git add objects/COD50000\ -\ My\ Codeunit.txt`

### Bij "setup"

Initialiseer git-structuur voor een nieuw C/AL project:

```bash
# Mapstructuur aanmaken
mkdir -p objects reports scripts

# .gitignore
cat > .gitignore << 'EOF'
*.fob
*.log
finsql.log
export_temp/
*.bak
EOF

# Initiële export van alle eigen objecten
# (gebruik /cal-export voor de daadwerkelijke export)
```

## Regels

- Exporteer altijd naar `.txt`, nooit `.fob` voor git
- Normaliseer datum/tijd altijd vóór commit
- Documenteer schema-wijzigingen in de commit message (SyncMode die nodig is)
- Eén logische wijziging per commit — niet alle gewijzigde objecten bulken
- Bij twijfel over conflict: importeer beide versies in een test-instantie en vergelijk gedrag
