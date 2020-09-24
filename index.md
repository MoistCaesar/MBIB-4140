# Index
- [Index](#index)
  - [Formål](#formål)
  - [Datakilder](#datakilder)
    - [Primære](#primære)
    - [Sekundære](#sekundære)
  - [Design](#design)
    - [Data: Postnummer](#data-postnummer)

## Formål
Det kan være interessant å sammenligne totalt antall innbyggere i en region mot hvor mange som tar høyere utdanning der og diverse andre sammenstillinger som kanskje kommer opp underveis. Dette vil da ikke nødvendigvis danne et hundre prosent korrekt bilde av situasjonen da det er flere faktorer (_fjernstudenter, pendlere, etc_) som spiller inn. Jeg har derfor valgt å se på data fra NSD og SSB i første rekke.

## Datakilder
### Primære
[**SSB**](https://data.norge.no/datasets/e74957b7-d052-4d93-9afb-4a2fce65882f) for demografisk informasjon per region. SSB er et naturlig valg for denne typen informasjon i Norge, og gir oss informasjon om demografi, kriminalitet, flytting og lignende. Data er tilgjengelig både i CSV og JSON.
[**NSD**](https://data.norge.no/datasets/9efe2de1-1093-4662-a8cb-fd7907bae9bc) tilbyr offentlig tilgjengelig data om høgskoler og universitet. Igjen, et svært naturlig og konvensjonelt valg. Her får vi diverse datasett, for eksempel oversikt over hvor mange har søkt om, er i, eller har fullført høyere utdanning. Data er tilgjengelig i CSV og JSON over **90** tabeller.

### Sekundære
Jeg vurderte [BaseBibliotek](https://www.nb.no/basebibliotek/?lang=no) som et alternativ. Etter å ha undersøkt datasettet de tilbyr er dette egentlig ikke av veldig stor interesse per i dag. Mulig kombineres med data fra NSD for å vise bittelitt informasjon om lokalt høgskole-/universitetsbibliotek.

## Design
### Data: NSD
Jeg tenker at jeg starter med data fra NSD. Hvilke(n) serie data som er interessant fra SSB er avhengig av hvilke data som kommer i NSDs sett. I første omgang er jeg interessert i følgende fra NSD:
- Antall studenter, kvinner & menn
- Institusjonens navn
- Studieprogrammets/instituttets navn
Disse variablene er tilgjengelige i tabell *123-Registrerte studenter* ([dokumentasjon](https://dbh.nsd.uib.no/dokumentasjon/tabell.action?tabellId=123)). Umiddelbart har jeg da et problem: for å lenke institusjonens navn til data for kommune/fylke må vi vite hvor skolene faktisk ligger. Jeg er kanskje også interessert i om fagskolen er privat eller offentlig. Dette finnes i tabell 535 eierforhold ([dokumentasjon](https://dbh.nsd.uib.no/dokumentasjon/tabell.action?tabellId=535)) samt tabell *211-Institusjon* ([dokumentasjon](https://dbh.nsd.uib.no/dokumentasjon/tabell.action?tabellId=211)), som gir oss adresseinformasjon. Her har vi også institusjonstype.

Eventuelle søk må da kunne (1) finne en/flere institusjoner, (2) innhente data, (3) hente og referere adresse/postnummerdata med en annen base, da NSD ikke har (maskinleselig) kommuneinformasjon i sine tabeller. Dermed tenker jeg at jeg kommer til å designe en spørring som (1) henter data fra NSD-tabellen(e), (2) sammenligner postnummer **A** med listen fra Posten, (3) returnerer sted **B** fra **A** og (4) anvender **B** for å hente stedsinformasjon fra SSB. Logisk? Ja, tror det. Raskt? Ikke nødvendigvis.

### Data: Posten/Bring
Heldigvis har Posten *(**Bring**)* [data om postnummer i Norge, dog i ANSI-format](https://www.bring.no/radgivning/sende-noe/adressetjenester/postnummer). Uheldigvis har Postens tabulatorseparerte ANSI-fil tegnsettfeil: ÆØÅ representeres som **?**. Fordi jeg planlegger å ikke bare vise informasjon fra filen men også bruke responsen som søkeord er dette en katastrofal feil. Heldigvis er denne tegnsettfeilen ikke tilstede i den alternative .xslx-filen, hvilket tillater meg å konvertere denne filen til en kommaseparert CSV uten tegnsettfeil. Dataen er strukturert som følger: 1 postkode, 2 sted, 3 fylkenummer og kommunenummer, 4 kommune, 5 postkodetype. For mine behov er jeg interessert i å søke gjennom kolonne 1 og hente data fra 4. Vi må observere at kolonne 3 har verdier som kan sammenfalle med 1, hvilket ekskluderer det aller enkleste søket ("finn 3001" returnerer ikke postkode 3001 Drammen, men 3001 Halden). Dersom vi ønsker fylkesstatistikk må vi enten søke på kommune i en SSB-tabell ~~eller anvende de første 2 tegnene i kolonne 3 for å oversette til søkestreng~~. Disse kodene er forøvrig innhentet fra SSB, og samsvarer med de som ellers anvendes i forvaltningen.

<details><summary>Fylkesnummer</summary>
03 Oslo, 11 Rogaland, 15 Møre og Romsdal, 18 Nordland, 21 Svalbard, 22 Jan Mayen, 30 Viken, 34 Innlandet, 38 Vestfold og Telemark, 42 Agder, 46 Vestland, 50 Trøndelag, 54 Troms og Finnmark.
</details>

### Data: SSB
