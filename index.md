# Index
- [Index](#index)
  - [Formål](#formål)
  - [Datakilder](#datakilder)
    - [Primære](#primære)
    - [Sekundære](#sekundære)
  - [Design](#design)
    - [Data: NSD](#data-nsd)
    - [Data: Posten/Bring](#data-postenbring)
    - [Data: SSB](#data-ssb)

## Formål
Det kan være interessant å sammenligne totalt antall innbyggere i en region mot hvor mange som tar høyere utdanning der og diverse andre sammenstillinger som kanskje kommer opp underveis. Dette vil da ikke nødvendigvis danne et hundre prosent korrekt bilde av situasjonen da det er flere faktorer (_fjernstudenter, pendlere, etc_) som spiller inn. Jeg har derfor valgt å se på data fra NSD og SSB i første rekke, selv om sammenstillinger av denne informasjonen svært sikkert allerede eksisterer.

## Datakilder
### Primære
[**SSB**](https://data.norge.no/datasets/e74957b7-d052-4d93-9afb-4a2fce65882f) for demografisk informasjon per region. SSB er et naturlig valg for denne typen informasjon i Norge, og gir oss informasjon om demografi, kriminalitet, flytting og lignende. Data er tilgjengelig både i CSV og JSON-STAT.

[**NSD**](https://data.norge.no/datasets/9efe2de1-1093-4662-a8cb-fd7907bae9bc) tilbyr offentlig tilgjengelig data om høgskoler og universitet. Igjen, et svært naturlig og konvensjonelt valg. Her får vi diverse datasett, for eksempel oversikt over hvor mange har søkt om, er i, eller har fullført høyere utdanning. Data er tilgjengelig i CSV og JSON over **90** tabeller.

### Sekundære
Jeg vurderte [BaseBibliotek](https://www.nb.no/basebibliotek/?lang=no) som et alternativ. Etter å ha undersøkt datasettet de tilbyr er dette egentlig ikke av veldig stor interesse per i dag. Mulig kombineres med data fra NSD for å vise bittelitt informasjon om lokalt høgskole-/universitetsbibliotek.

[NSR](https://data-nsr.udir.no//swagger/ui/index) kan være av interesse dersom vi vil inkludere utdanning på lavere trinn også.

## Design
Eventuelle søk må kunne (1) finne en/flere institusjoner, (2) innhente data, (3) hente og referere adresse/postnummerdata med en annen base, da NSD ikke har direkte kommuneinformasjon i tabell 211. Dermed tenker jeg at jeg kommer til å designe en spørring som (1) henter data fra NSD-tabellen(e), (2) sammenligner postnummer **A** med listen fra Posten, (3) returnerer sted **B** fra **A** og (4) anvender **B** for å hente stedsinformasjon fra SSB. Men kanskje jeg skulle gjøre det slik at man heller velger kommune, ikke en spesifikk utdanningsinstitusjon? Da vil prosessen reverseres: vi kommer da til å søke på en spesifikk kommune (via Postens datasett) i SSB, og henter samtlige postkoder i den kommunen for å søke gjennom NSDs tabell 211 gjennom en for-løkke. Vi må deretter matche resultatene fra 211 med tabell 123.

### Data: NSD
Format: JSON/CSV
I første omgang er jeg interessert i følgende fra NSD:
- Antall studenter, kvinner & menn
- Institusjonens navn
- Studieprogrammets/instituttets navn
Disse variablene er tilgjengelige i tabell *123-Registrerte studenter* ([dokumentasjon](https://dbh.nsd.uib.no/dokumentasjon/tabell.action?tabellId=123)). Umiddelbart har jeg da et problem: for å lenke institusjonens navn til data for kommune/fylke må vi vite hvor skolene faktisk ligger, hvilket ikke fremgår i denne tabellen. Dette finnes i tabell *211-Institusjon* ([dokumentasjon](https://dbh.nsd.uib.no/dokumentasjon/tabell.action?tabellId=211)), som også gir oss adresseinformasjon. Her har vi også institusjonstype.

<details><summary>Tabell 123 - Spørring</summary>
{"tabell_id":123,"api_versjon":1,"statuslinje":"N","begrensning":"10000","kodetekst":"J","desimal_separator":".",
"groupBy":["Institusjonskode", "Avdelingskode", "Årstall", "Semester", "Studentkategori"],
"sortBy":["Institusjonskode", "Avdelingskode"],
"filter":[
   {
      "variabel": "Institusjonskode",
      "selection": {
         "filter": "all",
         "values": [
            "*"
         ],
         "exclude": [
            ""
         ]
      }
   },
   {
      "variabel": "Avdelingskode",
      "selection": {
         "filter": "all",
         "values": [
            "*"
         ],
         "exclude": [
            ""
         ]
      }
   },
   {
      "variabel": "Årstall",
      "selection": {
         "filter": "top",
         "values": [
            "2"
         ],
         "exclude": [
            ""
         ]
      }
   }
]} 
</details>

<details><summary>Tabell 211 - Spørring</summary>
{"tabell_id":211,"api_versjon":1,"statuslinje":"N","begrensning":"10000","kodetekst":"J","desimal_separator":".",
"variabler":["*"],
"sortBy":["Institusjonskode"],
"filter":[
   {
      "variabel": "Institusjonskode",
      "selection": {
         "filter": "all",
         "values": [
            "*"
         ],
         "exclude": [
            ""
         ]
      }
   }
]}
</details>

### Data: Posten/Bring
Format: CSV
Heldigvis har Posten *(**Bring**)* [data om postnummer i Norge, dog i ANSI-format](https://www.bring.no/radgivning/sende-noe/adressetjenester/postnummer). *Uheldigvis* har Postens tabulatorseparerte ANSI-fil tegnsettfeil: ÆØÅ representeres som **?**. Fordi jeg planlegger å ikke bare vise informasjon fra filen men også bruke responsen som søkeord er dette en katastrofal feil. Heldigvis er denne tegnsettfeilen ikke tilstede i den alternative .xslx-filen, hvilket tillater meg å konvertere denne filen til en kommaseparert CSV uten tegnsettfeil. Dataen er strukturert som følger: 1 postkode, 2 sted, 3 fylkenummer og kommunenummer, 4 kommune, 5 postkodetype. Disse kodene er forøvrig innhentet fra SSB, og samsvarer med de som ellers anvendes i forvaltningen. For mine behov er jeg interessert i å søke gjennom kolonne 1 og hente data fra 4. Vi må observere at kolonne 3 har verdier som kan sammenfalle med 1, hvilket ekskluderer det aller enkleste søket (fritekst "finn 3001" returnerer ikke postkode 3001 Drammen, men 3001 Halden). Pandas fjerner ledende nuller i integer (eks. 0452 blir til 452). Vi må derfor reintrodusere disse til postkodene våre før vi kan anvende dem andre steder.

<details><summary>Fylkesnummer</summary>
03 Oslo, 11 Rogaland, 15 Møre og Romsdal, 18 Nordland, 21 Svalbard, 22 Jan Mayen, 30 Viken, 34 Innlandet, 38 Vestfold og Telemark, 42 Agder, 46 Vestland, 50 Trøndelag, 54 Troms og Finnmark.
</details>

### Data: SSB
Format: JSON-STAT/CSV
[Dokumentasjon for SSB](https://www.ssb.no/omssb/tjenester-og-verktoy/api/px-api/_attachment/248256?). SSB tilbyr flere APIer, hvorav én gir oss predefinerte datasett og en annen gir oss mulighet til å definere egne datasett. GET-protokoll fungerer kun for å hente hele datasett fra førstnevnte. For egendefinerte datasett skal POST anvendes. Fra SSB får vi respons i form av CSV eller JSON-Stat, en type JSON utformet for statistiske formål. Uten et bibliotek som kan tolke dette formatet er json-stat nesten uleselig. I Python kan vi bruke [pyjstat](https://pypi.org/project/pyjstat/) for å konvertere json-stat til pandas. Dette krever installasjon via PIP.

Vi skal i første omgang hente befolkningstall etter kjønn og alder knyttet til kommune. Det er da aktuelt å se på det predefinerte settet [577297 Befolkning, etter kjønn og ettårig alder. Kommuner, pr 1.1. siste år](http://data.ssb.no/api/v0/dataset/577297.csv?lang=no) - men ettårig alder er strengt tatt mye mer granulært enn det mine behov krever. Jeg peiler meg derfor inn på en tabell med femårig- eller tiårig alder som i settet [11727 Befolkning, etter kjønn og 10 funksjonelle aldersgrupper. Kommuner, pr. 1.1.2007 - siste år](http://data.ssb.no/api/v0/dataset/85699.csv?lang=no) eller et egendefinert sett. Etter noe eksperimentering bestemmer jeg meg for å gjøre sistnevnte.

Jeg henter [07459: Befolkning, etter region, kjønn, alder, statistikkvariabel og år](https://data.ssb.no/api/v0/no/table/07459/) fra SSB med alle kommuner og de tre siste år i CSV. API-spørring ville sett slik ut:
<details><summary>API-spørring</summary>
https://data.ssb.no/api/v0/no/table/07459/
{
  "query": [
    {
      "code": "Region",
      "selection": {
        "filter": "agg:KommSummer",
        "values": [
          "K-3001","K-3002","K-3003","K-3004","K-3005","K-3006","K-3007","K-3011","K-3012","K-3013","K-3014","K-3015","K-3016","K-3017","K-3018","K-3019","K-3020","K-3021","K-3022","K-3023","K-3024","K-3025","K-3026","K-3027","K-3028","K-3029","K-3030","K-3031","K-3032","K-3033","K-3034","K-3035","K-3036","K-3037","K-3038","K-3039","K-3040","K-3041","K-3042","K-3043","K-3044","K-3045","K-3046","K-3047","K-3048","K-3049","K-3050","K-3051","K-3052","K-3053","K-3054","K-0301","K-3401","K-3403","K-3405","K-3407","K-3411","K-3412","K-3413","K-3414","K-3415","K-3416","K-3417","K-3418","K-3419","K-3420","K-3421","K-3422","K-3423","K-3424","K-3425","K-3426","K-3427","K-3428","K-3429","K-3430","K-3431","K-3432","K-3433","K-3434","K-3435","K-3436","K-3437","K-3438","K-3439","K-3440","K-3441","K-3442","K-3443","K-3446","K-3447","K-3448","K-3449","K-3450","K-3451","K-3452","K-3453","K-3454","K-3801","K-3802","K-3803","K-3804","K-3805","K-3806","K-3807","K-3808","K-3811","K-3812","K-3813","K-3814","K-3815","K-3816","K-3817","K-3818","K-3819","K-3820","K-3821","K-3822","K-3823","K-3824","K-3825","K-4201","K-4202","K-4203","K-4204","K-4205","K-4206","K-4207","K-4211","K-4212","K-4213","K-4214","K-4215","K-4216","K-4217","K-4218","K-4219","K-4220","K-4221","K-4222","K-4223","K-4224","K-4225","K-4226","K-4227","K-4228","K-1101","K-1103","K-1106","K-1108","K-1111","K-1112","K-1114","K-1119","K-1120","K-1121","K-1122","K-1124","K-1127","K-1130","K-1133","K-1134","K-1135","K-1144","K-1145","K-1146","K-1149","K-1151","K-1160","K-4601","K-4602","K-4611","K-4612","K-4613","K-4614","K-4615","K-4616","K-4617","K-4618","K-4619","K-4620","K-4621","K-4622","K-4623","K-4624","K-4625","K-4626","K-4627","K-4628","K-4629","K-4630","K-4631","K-4632","K-4633","K-4634","K-4635","K-4636","K-4637","K-4638","K-4639","K-4640","K-4641","K-4642","K-4643","K-4644","K-4645","K-4646","K-4647","K-4648","K-4649","K-4650","K-4651","K-1505","K-1506","K-1507","K-1511","K-1514","K-1515","K-1516","K-1517","K-1520","K-1525","K-1528","K-1531","K-1532","K-1535","K-1539","K-1547","K-1554","K-1557","K-1560","K-1563","K-1566","K-1573","K-1576","K-1577","K-1578","K-1579","K-5001","K-5006","K-5007","K-5014","K-5020","K-5021","K-5022","K-5025","K-5026","K-5027","K-5028","K-5029","K-5031","K-5032","K-5033","K-5034","K-5035","K-5036","K-5037","K-5038","K-5041","K-5042","K-5043","K-5044","K-5045","K-5046","K-5047","K-5049","K-5052","K-5053","K-5054","K-5055","K-5056","K-5057","K-5058","K-5059","K-5060","K-5061","K-1804","K-1806","K-1811","K-1812","K-1813","K-1815","K-1816","K-1818","K-1820","K-1822","K-1824","K-1825","K-1826","K-1827","K-1828","K-1832","K-1833","K-1834","K-1835","K-1836","K-1837","K-1838","K-1839","K-1840","K-1841","K-1845","K-1848","K-1851","K-1853","K-1856","K-1857","K-1859","K-1860","K-1865","K-1866","K-1867","K-1868","K-1870","K-1871","K-1874","K-1875","K-5401","K-5402","K-5403","K-5404","K-5405","K-5406","K-5411","K-5412","K-5413","K-5414","K-5415","K-5416","K-5417","K-5418","K-5419","K-5420","K-5421","K-5422","K-5423","K-5424","K-5425","K-5426","K-5427","K-5428","K-5429","K-5430","K-5432","K-5433","K-5434","K-5435","K-5436","K-5437","K-5438","K-5439","K-5440","K-5441","K-5442","K-5443","K-5444","K-21-22","K-23","K-Rest"
        ]
      }
    },
    {
      "code": "Kjonn",
      "selection": {
        "filter": "item",
        "values": [
          "1",
          "2"
        ]
      }
    },
    {
      "code": "Alder",
      "selection": {
        "filter": "agg:TiAarigGruppering",
        "values": [
          "F00-09",
          "F10-19",
          "F20-29",
          "F30-39",
          "F40-49",
          "F50-59",
          "F60-69",
          "F70-79",
          "F80-89",
          "F90-99",
          "F100G10+"
        ]
      }
    },
    {
      "code": "Tid",
      "selection": {
        "filter": "item",
        "values": [
          "2018",
          "2019",
          "2020"
        ]
      }
    }
  ],
  "response": {
    "format": "csv2"
  }
}
</details>

Etter å ha lekt meg litt med denne dataen innser jeg jo at vi egentlig bare kan hente ut én kommune fra SSB. Hvorfor hente inn hele greia? Det vil bare gjøre ting tregere. Jeg lager da en dynamisk spørring ved å erstatte *alle* K-kodene i spørringen med variabelen kommuneTilSSB, som er informert av følgende: prefiksSSB = 'K-' kommuneID = *X* kommuneTilSSB = prefiksSSB + kommuneID. Jeg gjør det samme med år, så kan brukeren velge det selv. Vi får da en string tilbake som (ikke) behøver dekoding, og bruker io for å gjøre det til en df.

Loke Sjølie
08-10-2020