# C/AL Object Types & ID Management

---

## Objecttypen

| Type | Afkorting | ID range (custom) | Beschrijving |
|------|-----------|------------------|-------------|
| Table | TAB | 50000–99999 | Databastabellen |
| Form | FOR | 50000–99999 | **Oud** — vervangen door Page in NAV 2009 |
| Report | REP | 50000–99999 | Rapporten (RDLC layout) |
| Dataport | DAT | 50000–99999 | **Oud** — vervangen door XMLport in NAV 2009 |
| Codeunit | COD | 50000–99999 | Business logic |
| XMLport | XML | 50000–99999 | Import/export XML of CSV |
| MenuSuite | MEN | 50000–99999 | Navigatiemenu's |
| Page | PAG | 50000–99999 | UI pagina's (NAV 2009+) |
| Query | QUE | 50000–99999 | Data queries (NAV 2013+) |

**Standaard Microsoft-objecten**: ID 1–49999 (nooit aanpassen zonder migratiestrategie).
**Aanpassingen aan base-objecten**: dit is in C/AL normaal — je overschrijft het origineel.

---

## Structuur van een C/AL object (tekst-export formaat)

### Table
```
OBJECT Table 50000 My Table
{
  OBJECT-PROPERTIES
  {
    Date=01-01-24;
    Time=12:00:00;
    Modified=Yes;
    Version List=MYAPP1.00;
  }
  PROPERTIES
  {
    CaptionML=[ENU=My Table;
               NLD=Mijn Tabel];
  }
  FIELDS
  {
    { 1   ;   ;Code                ;Code20        ;CaptionML=[ENU=Code;NLD=Code] }
    { 2   ;   ;Name                ;Text100       ;CaptionML=[ENU=Name;NLD=Naam] }
    { 3   ;   ;Amount              ;Decimal       ;CaptionML=[ENU=Amount;NLD=Bedrag];
                                                   FieldClass=FlowField;
                                                   CalcFormula=Sum("My Line".Amount WHERE (Header No.=FIELD(Code))) }
  }
  KEYS
  {
    {    ;Code                    ;Clustered=Yes }
    {    ;Name                     }
  }
  CODE
  {
    BEGIN
    END.
  }
}
```

### Codeunit
```
OBJECT Codeunit 50000 My Codeunit
{
  OBJECT-PROPERTIES
  {
    Date=01-01-24;
    Time=12:00:00;
    Modified=Yes;
    Version List=MYAPP1.00;
  }
  PROPERTIES
  {
    OnRun=BEGIN
            // OnRun trigger
          END;
  }
  CODE
  {
    VAR
      gText@1000 : Text[100];

    PROCEDURE MyFunction@1(pValue : Integer) : Boolean;
    VAR
      lResult@1000 : Boolean;
    BEGIN
      EXIT(lResult);
    END;

    BEGIN
    END.
  }
}
```

### Page
```
OBJECT Page 50000 My Page
{
  OBJECT-PROPERTIES
  {
    Date=01-01-24;
    Time=12:00:00;
    Modified=Yes;
    Version List=MYAPP1.00;
  }
  PROPERTIES
  {
    CaptionML=[ENU=My Page;NLD=Mijn Pagina];
    SourceTable=Table50000;
    PageType=Card;
    OnOpenPage=BEGIN
                 // code
               END;
  }
  CONTROLS
  {
    { 1000;0;Container;ContainerType=ContentArea }
    { 1001;1;Group   ;CaptionML=[ENU=General;NLD=Algemeen] }
    { 1002;2;Field   ;SourceExpr=Code }
    { 1003;2;Field   ;SourceExpr=Name }
  }
  CODE
  {
    BEGIN
    END.
  }
}
```

---

## Version List

Elk object heeft een `Version List` veld — cruciaal voor C/AL objectbeheer:

```
Version List=NAVW19.00.00,MYAPP1.05;
```

- `NAVW19.00.00` = Microsoft base versie (NAV 2013 = W17, NAV 2018 = W19, BC14 = BC14.00)
- `MYAPP1.05` = eigen aanpassing, versie 1.05
- Meerdere tags gescheiden door komma
- Gebruik consistent voor traceerbaarheid

**Conventie voor eigen aanpassingen:**
- Prefix: bedrijfsafkorting of app-naam (bijv. `AVIKO`, `MYADD`)
- Versienummer: major.minor (bijv. `1.05`, `2.00`)

---

## Datum/Tijd in objectproperties

Het exportformaat gebruikt de regionale Windows-instelling van de server:
- Nederlands: `01-01-24` (DD-MM-YY)
- Engels: `01/01/24` (MM/DD/YY)

Dit veroorzaakt diff-ruis bij wisselende omgevingen. Oplossing: normaliseer naar één formaat in git via een pre-commit hook of export-script.

---

## Base object aanpassen vs. nieuw object

In C/AL zijn er twee strategieën:

**1. Base object aanpassen** (gebruikelijk in NAV/C/AL)
- Voordeel: eenvoudig, alles op één plek
- Nadeel: moeilijk te upgraden (conflicts met nieuwe Microsoft versie)
- Gebruik Version List om wijzigingen te markeren

**2. Event-gebaseerd patroon** (BC14 C/AL mode, vergelijkbaar met AL)
- Gebruik `[EventSubscriber]` in aparte codeunit
- Houd base objects schoon
- Vereist dat het base object events publiceert

**Richtlijn**: bij BC14 C/AL bij voorkeur event-gebaseerd werken om de migratie naar AL te vergemakkelijken.

---

## Relatie tot AL objecten (BC14)

In BC14 kunnen C/AL en AL objecten naast elkaar bestaan, maar:
- C/AL tabellen zijn zichtbaar voor AL via `tableextension`
- AL extensions kunnen C/AL events subscriben
- C/AL code kan AL objecten NIET aanroepen
- Vermijd ID-overlap: zorg dat AL en C/AL objecten verschillende ID-ranges gebruiken
