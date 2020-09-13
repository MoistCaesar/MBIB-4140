# Index
- [Index](#index)
  - [Formål](#formål)
  - [Datakilder](#datakilder)
    - [Primære](#primære)
    - [Sekundære](#sekundære)
  - [Design](#design)

## Formål
Det kan være interessant å sammenligne totalt antall innbyggere i en region mot hvor mange som tar høyere utdanning der og diverse andre sammenstillinger som kanskje kommer opp underveis. Dette vil da ikke nødvendigvis danne et hundre prosent korrekt bilde av situasjonen da det er flere faktorer (_fjernstudenter, pendlere, etc_) som spiller inn. Jeg har derfor valgt å se på data fra NSD og SSB i første rekke.

## Datakilder
### Primære
[**SSB**](https://data.norge.no/datasets/e74957b7-d052-4d93-9afb-4a2fce65882f) for demografisk informasjon per region. SSB er et naturlig valg for denne typen informasjon i Norge, og gir oss informasjon om demografi, kriminalitet, flytting og lignende. Data er tilgjengelig både i CSV og JSON.
[**NSD**](https://data.norge.no/datasets/9efe2de1-1093-4662-a8cb-fd7907bae9bc) tilbyr offentlig tilgjengelig data om høgskoler og universitet. Igjen, et svært naturlig og konvensjonelt valg. Her får vi diverse datasett, for eksempel oversikt over hvor mange har søkt om, er i, eller har fullført høyere utdanning. Data er tilgjengelig i CSV og JSON.

### Sekundære
Jeg vurderte [BaseBibliotek](https://www.nb.no/basebibliotek/?lang=no) som et alternativ. Etter å ha undersøkt datasettet de tilbyr er dette egentlig ikke av veldig stor interesse per i dag. Mulig kombineres med data fra NSD for å vise bittelitt informasjon om lokalt høgskole-/universitetsbibliotek.

## Design
