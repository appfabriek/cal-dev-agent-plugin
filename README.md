# cal-dev-agent-plugin

Claude Code plugin voor C/AL ontwikkeling in Business Central 14, NAV 2009–2018 en klassiek Navision. Werkt ook in Copilot CLI en Gemini CLI.

---

## Installatie

```bash
git clone https://github.com/appfabriek/cal-dev-agent-plugin.git ~/code/cal-dev-agent-plugin
```

In een Claude Code / Copilot / Gemini sessie:

```
/plugin install /pad/naar/cal-dev-agent-plugin
```

### Vereisten

- Self-hosted GitHub Actions runner op de BC/NAV-server (`bc-runner.yaml` van [bc-dev-agent-plugin](https://github.com/appfabriek/bc-claude-plugin))
- NavAdminTool beschikbaar op de runner (`NavAdminTool.ps1`)
- finsql.exe beschikbaar voor oudere NAV-versies zonder NavAdminTool

### Samenwerking met bc-dev-agent-plugin

Deze plugin is complementair aan de [bc-dev-agent-plugin](https://github.com/appfabriek/bc-claude-plugin). De runner-infrastructuur (`bc-runner.yaml`) en NavAdminTool-kennis worden gedeeld. Installeer beide voor volledige ondersteuning van projecten met zowel C/AL als AL.

---

## Skills (7 commands)

| Commando | Doel |
|----------|------|
| `/cal-init <database>` | Nieuw project opstarten: instance aanmaken, structuur aanleggen, initiële export |
| `/cal-pull` | Haal laatste code op uit database — voor klanten die wijzigingen in DB laten staan |
| `/cal-export [filter]` | Exporteer specifieke C/AL objecten uit de database naar `.txt` bestanden |
| `/cal-import [bestand]` | Importeer en compileer C/AL objecten in de database |
| `/cal-git [actie]` | Git-beheer: exporteren voor commit, diffs, merge-conflicten |
| `/cal-review [object]` | Review C/AL code op kwaliteit, correctheid en valkuilen |
| `/cal-deploy [omgeving]` | Deploy gewijzigde C/AL objecten naar omgevingen |

---

## Knowledge Base (4 bestanden)

| Bestand | Inhoud |
|---------|--------|
| `cal-guidelines.md` | C/AL syntax, triggers, datatypes, veelgemaakte fouten, performance |
| `cal-objects.md` | Objecttypen, ID-beheer, tekst-exportformaat, Version List |
| `cal-tools.md` | Export-/Import-NAVApplicationObject, finsql.exe, Get-NAVApplicationObject |
| `cal-git-patterns.md` | Git-strategie voor C/AL, naamgeving, diff lezen, merge-aanpak |

---

## Ondersteunde platforms

| Platform | Versie | Status |
|----------|--------|--------|
| Business Central 14 | 14.x on-prem | Volledig ondersteund |
| NAV 2018 | 11.x | Volledig ondersteund |
| NAV 2016–2017 | 9.x–10.x | Ondersteund |
| NAV 2013–2015 | 7.x–8.x | Ondersteund |
| NAV 2009 | 6.x | Ondersteund (finsql fallback) |
| Navision 5.x en ouder | 5.x | Gepland |

---

## Workflow

```
0. /cal-init <database>      ← eenmalig: project aanmaken + eerste export
   (daarna regelmatig:)
1. /cal-pull                 ← haal laatste code op uit DB (klanten wijzigen vaak direct)
2. Bewerk in teksteditor
3. /cal-review               ← controleer vóór import
4. /cal-import               ← compileer in DB
5. Test in client
6. /cal-git commit           ← commit naar git
7. /cal-deploy [omgeving]    ← deploy naar andere omgevingen
```

---

## Verschil met bc-dev-agent-plugin

| | cal-dev-agent-plugin | bc-dev-agent-plugin |
|---|---|---|
| Taal | C/AL | AL |
| Broncode | In database | `.al` bestanden |
| Tooling | NavAdminTool + finsql | alc.exe + VS Code |
| Git | Export-eerst | Native |
| Platforms | BC14, NAV, Navision | BC15+ |
