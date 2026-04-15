# C/AL Guidelines

Richtlijnen voor C/AL ontwikkeling in BC14, NAV 2009–2018 en klassiek Navision.

---

## Syntax — Kernregels

### Variabelen
```cal
// Declaratie in VAR sectie van trigger of procedure
lText    : Text[100];
lCode    : Code[20];
lInt     : Integer;
lDec     : Decimal;
lBool    : Boolean;
lDate    : Date;
lTime    : Time;
lDT      : DateTime;
lRec     : Record "Customer";
lPage    : Page "Customer Card";
lReport  : Report "Customer - Order Summary";
lCdu     : Codeunit "Sales-Post";
```

Prefixen (zelfde conventie als AL waar van toepassing):
- `l` = local, `g` = global, `p` = parameter
- Typeaanduidingen: `Rec`, `Cdu`, `Txt`, `Int`, `Dec`, `Bln`, `Dt`

### Triggers vs Procedures
```cal
// Trigger — geen parameters, ingebouwd in object
OnValidate()
BEGIN
  // code
END;

// Procedure — handmatig aangemaakt, kan parameters en return value hebben
PROCEDURE MyProcedure(pValue : Integer) lResult : Boolean;
BEGIN
  // code
  EXIT(TRUE);
END;
```

### Recordoperaties
```cal
// Lezen met SETRANGE / SETFILTER
lRec.RESET;
lRec.SETRANGE("No.", '10000', '20000');
lRec.SETFILTER(Name, '*Smith*');
IF lRec.FINDSET THEN
  REPEAT
    // verwerk lRec
  UNTIL lRec.NEXT = 0;

// Lezen — enkel record
lRec.GET('10000');        // fout als niet gevonden
IF lRec.GET('10000') THEN // veilig

// Schrijven
lRec.INIT;
lRec."No." := '99999';
lRec.Name := 'Test';
lRec.INSERT(TRUE);       // TRUE = triggers uitvoeren

lRec.Name := 'Gewijzigd';
lRec.MODIFY(TRUE);

lRec.DELETE(TRUE);

// Locken voor update
lRec.LOCKTABLE;
IF lRec.GET('10000') THEN BEGIN
  lRec.Name := 'Nieuw';
  lRec.MODIFY;
END;
```

### Foutafhandeling
```cal
// Fout gooien
ERROR('Foutmelding %1', lValue);

// Bevestiging
IF NOT CONFIRM('Weet u het zeker?', FALSE) THEN
  EXIT;

// Melding
MESSAGE('Klaar. %1 records verwerkt.', lCount);

// Stille annulering (geen foutmelding)
ERROR('');
```

### Tekst en opmaak
```cal
// Concatenatie
lText := 'Hallo ' + lName;

// Formattering
lText := STRSUBSTNO('Nr: %1, Naam: %2', lNo, lName);

// Paddingt / opmaak
lText := FORMAT(lDate, 0, '<Day,2>-<Month,2>-<Year4>');
```

---

## Object triggers — volgorde

### Table triggers
```
OnInsert → OnModify → OnDelete → OnRename
OnValidate (per veld) → OnLookup (per veld)
```

### Page triggers
```
OnInit → OnOpenPage → OnAfterGetRecord → OnAfterGetCurrRecord
OnNewRecord → OnInsertRecord → OnModifyRecord → OnDeleteRecord
OnQueryClosePage → OnClosePage
```

### Codeunit triggers
```
OnRun
```

---

## Veelgemaakte fouten in C/AL

### Infinite loops via VALIDATE
```cal
// FOUT: OnValidate van veld A roept VALIDATE op veld B aan,
// veld B roept VALIDATE op veld A aan → infinite loop

// OPLOSSING: guard flag gebruiken
IF NOT gBoolSkipValidation THEN BEGIN
  gBoolSkipValidation := TRUE;
  VALIDATE("Related Field", NewValue);
  gBoolSkipValidation := FALSE;
END;
```

### FINDSET zonder NEXT
```cal
// FOUT: record pointer staat na FINDSET niet automatisch op eerste record
IF lRec.FINDSET THEN
  lRec.Name := 'Test'; // verwerkt alleen het eerste record — NEXT vergeten!

// GOED:
IF lRec.FINDSET THEN
  REPEAT
    lRec.Name := 'Test';
    lRec.MODIFY;
  UNTIL lRec.NEXT = 0;
```

### LOCKTABLE te laat
```cal
// FOUT: LOCKTABLE na GET → deadlock risico
lRec.GET('10000');
lRec.LOCKTABLE; // te laat

// GOED:
lRec.LOCKTABLE;
lRec.GET('10000');
```

### Tekstvelden afkappen
```cal
// Text[50] veld — lText is Text[100]
lRec."Short Field" := COPYSTR(lText, 1, MAXSTRLEN(lRec."Short Field"));
```

---

## FlowFields en CALCFIELDS

FlowFields bevatten geen data — ze worden berekend bij opvragen:

```cal
// FOUT: FlowField lezen zonder CALCFIELDS
IF lCust."Balance (LCY)" > 1000 THEN ...  // altijd 0!

// GOED:
lCust.CALCFIELDS("Balance (LCY)");
IF lCust."Balance (LCY)" > 1000 THEN ...

// In een loop — eenmalig na FINDSET
IF lCust.FINDSET THEN
  REPEAT
    lCust.CALCFIELDS("Balance (LCY)");
    // verwerk
  UNTIL lCust.NEXT = 0;
```

---

## Performance

- Gebruik `SETLOADFIELDS` niet — bestaat niet in C/AL. Minimaliseer velden die je gebruikt door SETRANGE/SETFILTER zo specifiek mogelijk te maken.
- Gebruik `FINDSET` i.p.v. `FIND('-')` bij meerdere records.
- Gebruik `FIND('-')` alleen als je het eerste record wilt.
- `LOCKTABLE` zo laat mogelijk, zo snel mogelijk vrij.
- Vermijd `FINDSET` → `MODIFY` in een grote loop zonder expliciete transactiegrenzen.
- Gebruik `COMMIT` bewust — C/AL commits automatisch aan het einde van een codeunit run.

---

## Naamgeving (NAV conventie)

| Type | Prefix | Voorbeeld |
|------|--------|-----------|
| Global variable | g | gCduSales |
| Local variable | l | lRecCustomer |
| Parameter | p | pNo |
| Boolean | Bool/Bln | gBoolSkipValidation |
| Codeunit | Cdu | lCduPost |
| Record | Rec / tabelnaam | lRecCust / lCustomer |

Object namen: beschrijvend, geen afkortingen waar mogelijk. Gebruik aanhalingstekens voor namen met spaties: `Record "Sales Header"`.

---

## Versieverschillen die de syntax beïnvloeden

| Feature | Beschikbaar vanaf |
|---------|-----------------|
| `QUERY` objecttype | NAV 2013 |
| `RECORDREF` / `FIELDREF` | NAV 2009 |
| `DOTNET` variabelen | NAV 2009 (beperkt) / NAV 2013+ (volledig) |
| `XMLPORT` vervangt `DATAPORT` | NAV 2009 |
| `PAGE.RUN` / `PAGE.RUNMODAL` | NAV 2009 |
| `TASKSCHEDULER` | NAV 2016+ |
| `SESSIONSETTINGS` | NAV 2018 / BC14 |
| `REPORT.RUNREQUESTPAGE` | NAV 2013+ |
