# Testing

[← Tilbake til README](../README.md)

Vi har fire typer tester. Enhetstestene kjører automatisk. System-, sikkerhets- og mobiltesten er kjørt manuelt i nettleseren mot appen i Docker, med MariaDB og ekte data. Brukervennlighetstesten gjøres med personer utenfor gruppa (se [status](#brukervennlighetstest)).

| Type | Hva | Hvor |
|---|---|---|
| Enhetstesting | Automatiske xUnit-tester av controllere, tjenester, repositoryer, modeller, validering og sikkerhet | [Automatiske tester](#automatiske-tester-xunit) |
| Systemtesting | Hovedflytene fra start til slutt, med alle tre rollene | [Systemtest](#systemtest) |
| Sikkerhetstesting | Forsøk på å bryte tilgangskontroll, CSRF, XSS, SQL-injeksjon, filopplasting og innlogging | [Sikkerhetstest](#sikkerhetstest) |
| Brukervennlighetstesting | Personer utenfor gruppa løser oppgaver i appen | [Brukervennlighetstest](#brukervennlighetstest) |

✅ bestått · ⚠️ bestått med merknad · ❌ feilet

## Automatiske tester (xUnit)

Testene kjører uten Docker og uten database. De bruker de minnebaserte repositoriene (`InMemory*Repository`), og Identity erstattes med NSubstitute.

```bash
dotnet test Beredskapsportal.Tests
```

**Resultat (05.11.2026):** 162 av 162 tester bestått ✅ (77 fra Oppgave 1 og 85 nye i Oppgave 2)

| Testfil | Hva den sjekker |
|---|---|
| `BehovControllerTester.cs` | POST av behov: gyldig skjema lagres, tomt skjema og manglende plassering avvises, tekst lagres uendret |
| `RessursControllerTester.cs` | POST av ressurs: lagres som «venter», ugyldig skjema lagres ikke, bilderegler (antall, størrelse, type, innhold), `/Ressurs/Bilde` kan ikke hente andre filer |
| `RessursListeTester.cs` | `/Ressurs`: private ser bare egne ressurser, offentlige godkjente og sine egne (også de som venter), admin alle, tildelte ressurser merkes, og lista sorteres med tilgjengelige først og avviste sist |
| `RedigerBehovTester.cs` | Endre behov: bare den som meldte det og admin, endringer lagres, status, dato og eier endres ikke, endret prioritet i historikken |
| `RedigerRessursTester.cs` | Endre ressurs: bare eier og admin, endringer lagres, ny godkjenning etter endring, nye bilder erstatter gamle, historikk |
| `ValideringTester.cs` | Valideringsreglene i skjemaene for behov, ressurs, innlogging og registrering (påkrevde felt, telefon, e-post, maks lengde) |
| `KartControllerTester.cs` | `/Kart/Behov` og `/Kart/Ressurser`: feltene kart.js bruker, bare godkjente ressurser, tildelte merket, bare GET og bare offentlige aktører |
| `KontoControllerTester.cs` | Innlogging (riktig og feil passord, ukjent bruker, låst konto, ikke godkjent), registrering, og endring av navn, e-post og passord på Min side |
| `OversiktControllerTester.cs` | «Mine meldte behov» på Oversikt |
| `SikkerhetTester.cs` | Sikkerhetsheadere og CSP, roller, norske feilmeldinger, rate limiting (også på Min side), grense på opplasting, sjekk av bildeinnhold |
| `TilgangTester.cs` | Hvilke roller controllerne krever |
| `DetaljerTester.cs` | Detaljsidene, og at private aktører bare ser sine egne ressurser |
| `TildelingTjenesteTester.cs`, `TildelingsforslagTester.cs` | Godkjenning, tildeling, forespørsler, tilbaketrekking og avstand |
| `ForesporselControllerTester.cs`, `TildelingControllerTester.cs`, `HvaTrengsControllerTester.cs` | Forespørsler, tildelingslista og «Hva trengs» |
| `BehovRepositoryTester.cs`, `TildelingRepositoryTester.cs`, `ModellTester.cs`, `ControllerTester.cs`, `IdentitetOppsettTester.cs` | Tellinger, eksempeldata, passordregler, forsiden og oppsett av roller og testbrukere |

`TestHjelpere.cs` og `IdentitetHjelpere.cs` inneholder erstatninger for innlogging, filsystem og Identity, så controllerne kan testes uten at appen kjører.

## Systemtest

**Dato:** 03.11.2026 · **Miljø:** Windows, Docker Desktop (app og MariaDB i Docker), Chrome · **Kode:** branchen `feature/oppgave2-endre-ressurs-og-testing` · **Testbrukere:** `test`, `privat` og `admin` (se [docs/drift.md](drift.md#testbrukere))

### Registrering og innlogging

| Scenario | Steg | Forventet | Faktisk |
|---|---|---|---|
| Registrere privat aktør | `/Konto/Registrer`, rolle privat aktør | Logges inn og sendes til Oversikt | ✅ Logget inn, navnet vises i menyen |
| Registrere offentlig aktør | `/Konto/Registrer`, rolle offentlig aktør | Kontoen lages, men må godkjennes | ✅ «Takk! Kontoen er opprettet, men må godkjennes av en administrator før du kan logge inn.» |
| Logge inn før godkjenning | Logg inn med den nye offentlige aktøren | Nektes | ✅ «Kontoen din venter på godkjenning fra en administrator.» |
| Admin godkjenner bruker | `/Admin` → «Godkjenn» | Brukeren kan logge inn og ser kartet | ✅ Logget inn, `/Kart` åpnes |
| Brukernavn opptatt | Registrer `test` | Feilmelding | ✅ «Brukernavnet «test» er allerede i bruk.» |
| Svakt passord | Passord uten tall og stor bokstav | Feilmelding | ✅ «Passordet må inneholde minst ett tall. Passordet må inneholde minst én stor bokstav.» |
| Feil passord | Riktig brukernavn, feil passord | Feilmelding | ✅ «Feil brukernavn eller passord.» |
| Tomt innloggingsskjema | Trykk «Logg inn» | Feilmelding ved begge felt | ✅ |
| Logge inn med de tre rollene | `test`, `privat`, `admin` | Sendes til Oversikt | ✅ |
| Min side uten innlogging (04.11.2026) | Åpne `/Konto/EndreProfil` og `/Konto/EndrePassord` uinnlogget | Sendes til innlogging | ✅ |
| Endre navn (04.11.2026) | Ny privat aktør → Min side → «Endre profil», nytt navn | Lagres, nytt navn vises med en gang | ✅ «Profilen er oppdatert.», navnet vises i menyen |
| Endre e-post uten / med feil passord (04.11.2026) | Ny e-post uten nåværende passord, så med feil passord | Avvises | ✅ «Skriv inn nåværende passord for å endre e-posten.» og «Nåværende passord er feil.» |
| Endre til e-post som er i bruk (04.11.2026) | E-posten til en annen bruker | Avvises | ✅ «E-posten «…» er allerede registrert.» |
| Endre e-post (04.11.2026) | Ny e-post med riktig passord | Lagres | ✅ Ny e-post vises på Min side |
| Endre passord, feil input (04.11.2026) | Feil nåværende passord · nye passord ulike · for kort passord | Feilmelding for hver | ✅ «Nåværende passord er feil.» · «Passordene er ikke like.» · «Passordet må være på minst 8 tegn.» |
| Endre passord (04.11.2026) | Riktig nåværende og gyldig nytt passord, logg ut og inn igjen | Fortsatt innlogget etter byttet, gammelt passord virker ikke, nytt virker | ✅ «Passordet er endret.», gammelt passord gir «Feil brukernavn eller passord.», nytt sender til Oversikt |

### Behov (offentlig aktør)

| Scenario | Steg | Forventet | Faktisk |
|---|---|---|---|
| Melde behov | Fyll ut `/Behov/Registrer` med punkt i kartet | Lagres, vises i `/Behov`, på kartet og på detaljsiden | ✅ Lagret i MariaDB, vises alle tre steder, historikk «Behovet ble meldt inn» |
| Tomt behovsskjema | Send uten å fylle ut | Feilmelding for hvert påkrevde felt | ✅ Type, beskrivelse, område, kontaktperson, telefon og plassering |
| For langt telefonnummer | 31 siffer | Feilmelding, ikke databasefeil | ✅ «Telefonnummeret kan være på maks 30 tegn.» |
| Endre eget behov (04.11.2026) | Detaljside → «Endre behovet», bytt beskrivelse og prioritet moderat → høy, lagre | Lagres med en gang, vises i historikken | ✅ «Endringene er lagret.», merket «Høy prioritet», historikk «Behovet ble endret, prioritet moderat → høy» |
| Endre andres behov (04.11.2026) | Annen offentlig aktør og privat aktør: GET og POST til `/Behov/Rediger/{id}` | Nektes, ingen «Endre»-knapp | ✅ Sendt til Oversikt, behovet uendret, knappen vises ikke |
| Admin endrer behov (04.11.2026) | Som admin: endre et behov meldt av `test` | Lagres, eier uendret | ✅ Historikken viser «av admin», behovet står fortsatt som meldt av `test` |
| Kartdata | `/Kart/Behov` og `/Kart/Ressurser` | JSON med feltene kartet bruker | ✅ 13 behov og 28 godkjente ressurser, 5 merket som tildelt. 41 markører på kartet |

### Ressurser (privat aktør og admin)

| Scenario | Steg | Forventet | Faktisk |
|---|---|---|---|
| Tilby ressurs | Fyll ut «Tilby ressurs» med et PNG-bilde | Lagres som «Venter på godkjenning» | ✅ «Takk! Ressursen er sendt inn, og blir synlig når en administrator har godkjent den.» |
| Ressurs uten bilde | Send uten bilde | Feilmelding | ✅ «Du må legge ved minst ett bilde av ressursen.» |
| Admin godkjenner ressurs | `/Admin` → godkjenn | Vises på kartet, bildet kan åpnes | ✅ |
| Endre ressurs uten nye bilder | Detaljside → «Endre ressursen», bytt type og beskrivelse, flytt punktet, lagre | Lagres, må godkjennes på nytt | ❌ → ✅ Første forsøk: nettleseren krevde nye bilder (se [Funnet under testing](#funnet-under-testing)). Etter rettingen: lagret, status «Venter på godkjenning», historikk «Ressursen ble endret og venter på ny godkjenning» |
| Endre ressurs med nytt bilde | Last opp nytt bilde | Nytt bilde erstatter det gamle | ✅ Ett bilde, det gamle gir 404 |
| Trekke tilbake ressurs | Detaljside → «Trekk tilbake» → bekreft | Ressursen og bildene slettes | ✅ Detaljsiden og bildet gir 404 |
| Data blir liggende | Bygg og start containerne på nytt (uten `-v`) | Ressursen finnes fortsatt | ✅ |
| Ressurslista (05.11.2026) | Som `test`: «Ressurser» i menyen. Som `admin`: det samme. Som `privat`: fanene Oversikt · Mine ressurser · Tilby ressurs · Hva trengs. Prøv filtrene, trykk på en ressurs, og sjekk 375 px bredde i DevTools | `test` ser godkjente med status «Tildelt» og dato, tilgjengelige øverst og tildelte under. `admin` ser alle, også de som venter og er avvist, og de avviste står nederst. Ingen ser hvem som tilbød ressursen i lista. `privat` ser bare type, adresse og status («Tildelt et behov») og knappen «Endre ressursen». Ingen feil i konsollen, og ingen sidelengs rulling på mobil | ✅ |

### Docker

| Scenario | Steg | Forventet | Faktisk |
|---|---|---|---|
| Starte app og database | `docker compose up --build` | To containere starter, appen svarer på http://localhost:8080 | ⚠️ Port 3306 var reservert av Windows. Rettet med `DB_PORT` i `.env` (se [docs/drift.md](drift.md)), deretter ✅ |

## Sikkerhetstest

Kjørt manuelt samme dag, med de samme brukerne. Forsøkene er gjort fra nettleseren med `fetch`, slik en angriper ville gjort, i tillegg til vanlige klikk.

| Angrep / kontroll | Hva vi gjorde | Forventet | Faktisk |
|---|---|---|---|
| Sikkerhetsheadere | Leste svaret fra `/` | CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy. Ingen `Server`-header | ✅ Alle på plass, `script-src 'self'` uten `unsafe-inline` |
| Uinnlogget tilgang | Åpnet `/Oversikt`, `/Kart`, `/Kart/Behov`, `/Kart/Ressurser`, `/Admin`, `/Behov`, `/Ressurs/Rediger/1` | Sendes til innlogging | ✅ Alle 7 |
| Feil rolle | Som privat aktør: `/Kart`, `/Behov`, `/Admin`, `/Kart/Ressurser`, andres ressurs og `/Ressurs/Rediger/1` | Sendes til Oversikt | ✅ Alle 6 |
| Endre andres ressurs | Annen privat bruker: GET og POST direkte til `/Ressurs/Rediger/{id}`, og åpne bildet | Nektes, ingenting endres | ✅ Nektet, ressursen uendret |
| Manipulere skjemaet | Sendte `Status=Godkjent`, `RegistrertAv=admin` og `EksisterendeBilder=../../Program.cs` med «Endre ressurs» | Ignoreres | ✅ Status fortsatt «Venter», eier uendret |
| CSRF | POST til `/Behov/Registrer` og `/Konto/LoggInn` uten anti-forgery-token | Avvises | ✅ 400 Bad Request |
| XSS | Beskrivelse med `<img src=x onerror=...>` og `<script>` | Vises som tekst, kjøres ikke | ✅ Vist som tekst på detaljsiden, i kartlista og i JSON. Sidetittelen ble ikke endret |
| SQL-injeksjon | Beskrivelse med `'); DROP TABLE Behov; --` | Lagres som tekst | ✅ Lagret uendret, tabellen `Behov` virker som før |
| Falskt bilde | Tekstfil kalt `falsk.jpg`, Windows-program (`MZ`) kalt `program.png` | Avvises | ✅ ««falsk.jpg» er ikke et gyldig JPG-bilde.» og ««program.png» er ikke et gyldig PNG-bilde.» |
| Stitraversering | `/Ressurs/Bilde/..%2F..%2Fappsettings.json` | Ingen fil sendes | ✅ 404 |
| Utestenging | 5 feil passord, så riktig passord | Kontoen låses | ✅ «For mange feil forsøk. Kontoen er låst i 5 minutter.» |
| Rate limiting | 14 innloggingsforsøk på under ett minutt fra samme IP | Fra forsøk 11: 429 | ✅ Forsøk 1–10 slapp gjennom, 11–14 fikk 429 «For mange forsøk på kort tid. Vent ett minutt og prøv igjen.» Vanlige sider virket som før |

## Mobiltest

Chrome-vinduet satt til 500 px bredt (det smaleste Chrome tillater), som er under Bootstraps mobilgrense på 576 px.

| Side | Forventet | Faktisk |
|---|---|---|
| Forside | Ingen sidelengs rulling | ✅ |
| Oversikt | Kortene stables i én kolonne | ✅ |
| Kart | Kartet fyller skjermen, «Filter» og «Vis liste» kan nås | ✅ |
| Melde behov, Tilby ressurs | Skjema og kart i én kolonne | ✅ |
| Behov (liste) | Hele tabellen kan leses | ⚠️ Tabellen rulles sidelengs inne i kortet. Kolonnene «Status» og «Dato» er skjult til man ruller, og ingenting viste at det gikk an. Rettet med teksten «Sveip sidelengs for å se alle kolonnene →» over tabeller på smal skjerm, sjekket 04.11.2026 ✅ |
| Detaljside for behov | Kartet under teksten | ✅ |
| Tildelinger, Hva trengs | Ingen sidelengs rulling | ✅ |

## Brukervennlighetstest

> **Delvis gjennomført:** rollen privat aktør er testet med én person (12.11.2026). Offentlig aktør og admin gjenstår.

**Hvordan:** Testpersonen fikk én oppgave om gangen, uten hjelp, og tenkte høyt mens hun løste den. Etter hver oppgave ga hun en karakter fra 1 (veldig vanskelig) til 5 (veldig lett). Til slutt svarte hun på noen spørsmål om hva som var lett, vanskelig og manglet. Én i gruppa observerte og noterte.

**Testpersoner:** Person A, 24 år, jobber i barnevernstjenesten, vant med apper, har ikke sett appen før, kjenner en i gruppa · **Dato:** 12.11.2026 · **Enhet:** Mac, Safari

| # | Oppgave | Rolle | Person A | Person B | Person C |
|---|---|---|---|---|---|
| 1 | Lag en bruker som privat aktør | – | Klarte alene (5/5) | | |
| 2 | Tilby et aggregat med bilde, og vis hvor det står | Privat aktør | Klarte alene (5/5) | | |
| 3 | Endre beskrivelsen av ressursen du la inn | Privat aktør | Klarte alene (5/5) | | |
| 4 | Meld et behov for nødstrøm på Lund med høy prioritet | Offentlig aktør (`test`) | Ikke testet | | |
| 5 | Finn alle ledige aggregater på kartet | Offentlig aktør | Ikke testet | | |
| 6 | Finn ut hvilke ressurser som er tildelt behovet ditt | Offentlig aktør | Ikke testet | | |
| 7 | Godkjenn en ressurs som venter, og tildel den til et behov | Admin | Ikke testet | | |
| 8 | Finn ressursen du la inn, og se hvilken status den har | Privat aktør | Klarte alene (5/5) | | |
| 9 | Finn ut hva som trengs akkurat nå i området ditt | Privat aktør | Klarte det, med nøling (5/5) | | |

Tallet i parentes er testpersonens egen karakter fra 1 (veldig vanskelig) til 5 (veldig lett).

| Funn | Hvor mange | Hva vi endret |
|---|---|---|
| Uklart hvem man skal kontakte, og hva man gjør videre, fra «Hva trengs». Nølte med å finne området nærmest seg | 1 av 1 | Ikke endret ennå |
| Savnet informasjon om hva som skjer med ressursen etter at den er meldt inn | 1 av 1 | Ikke endret ennå |
| «Venter på godkjenning» og «Tildelt et behov» ble forstått riktig | 1 av 1 | Ingen endring nødvendig |
| Ville brukt appen fordi det er «kjekt å hjelpe», men spurte også «hva får en igjen for å bidra?» | 1 av 1 | Ikke en feil i appen. Tas med som innspill |

**Tidligere tilbakemeldinger:** Etter fremføringen og sprint review endret vi blant annet lista under kartet til fullskjerm, la til filter på kartet og laget siden «Hva trengs» (commit `0d355f4`, «Rettelser etter fremføring»).

## Funnet under testing

| Feil | Hvor | Status |
|---|---|---|
| «Endre ressurs» kunne ikke lagres uten å velge nye bilder. Nettleseren viste «The Bilder av ressursen field is required.» Årsak: `List<IFormFile>` uten `?` får en skjult `[Required]` fra ASP.NET | `/Ressurs/Rediger` | Rettet: `Bilder` er nå `List<IFormFile>?`, testet på nytt ✅ |
| Port 3306 var reservert av Windows/Hyper-V, så databasen startet ikke | `docker-compose.yml` | Rettet: porten kan settes med `DB_PORT` i `.env` |
| Tekstfeltene hadde ikke maks lengde, mens kolonnene i databasen har det. For lang tekst ville gitt databasefeil | Alle skjemaer | Rettet: `[StringLength]` på feltene, testet ✅ |
| Tabellene viste ikke at de kan rulles sidelengs på mobil | `/Behov`, `/Tildeling`, `/Admin` | Rettet: tekst over tabellen på skjermer under 768 px (`site.css`) |
| Meldingen om manglende plassering ble vist to ganger (Oppgave 1) | `/Behov/Registrer`, `/Ressurs/Registrer` | Rettet i Oppgave 1, sjekket igjen i `ValideringTester` ✅ |

<details>
<summary>Systemtest fra Oppgave 1 (30.09.2026)</summary>

Kjørt før Identity, MariaDB og godkjenning. Testbruker `test` / `test`.

### Registrere bruker og logge inn/ut

| Scenario | Steg | Forventet resultat | Faktisk resultat |
|---|---|---|---|
| Registrere ny bruker | Gå til `/Konto/Registrer`, fyll ut alle felt og trykk «Registrer» | Brukeren logges inn og sendes til Oversikt | ✅ Logget inn og sendt til Oversikt. Navnet vises øverst til høyre |
| Logge ut | Trykk på navnet øverst til høyre | Brukeren sendes til forsiden, og menyen viser «Logg inn» | ✅ Som forventet |
| Logge inn | Logg inn med testbrukeren og med den nye brukeren | Brukeren sendes til Oversikt | ✅ Som forventet for begge brukerne |

### Feil passord og tomme felt (validering)

| Scenario | Steg | Forventet resultat | Faktisk resultat |
|---|---|---|---|
| Feil passord | Logg inn med riktig brukernavn og feil passord | Feilmelding, og brukeren blir på innloggingssiden | ✅ «Feil brukernavn eller passord.» |
| Tomt innloggingsskjema | Trykk «Logg inn» uten å fylle ut noe | Feilmelding ved hvert felt | ✅ «Du må skrive inn brukernavn.» og «Du må skrive inn passord.» |
| Tomt registreringsskjema | Trykk «Registrer» uten å fylle ut noe | Feilmelding ved hvert påkrevde felt | ✅ Feilmelding for alle fem feltene, øverst og under hvert felt |
| Ulike passord | Skriv to forskjellige passord ved registrering | Feilmelding | ✅ «Passordene er ikke like.» |
| Brukernavnet er tatt | Registrer en bruker med brukernavnet `test` | Feilmelding | ✅ «Brukernavnet er allerede i bruk.» |
| Tomt behovsskjema | Trykk «Registrer» på `/Behov/Registrer` uten å fylle ut noe | Feilmelding ved hvert påkrevde felt, også manglende posisjon | ⚠️ Alle feltene får feilmelding, men posisjonsmeldingen vises to ganger (se under) |

### Melde behov og registrere ressurs

| Scenario | Steg | Forventet resultat | Faktisk resultat |
|---|---|---|---|
| Melde behov | Fyll ut `/Behov/Registrer`, klikk i kartet og trykk «Registrer» | Brukeren sendes til `/Behov`, og det nye behovet står øverst | ✅ Behovet står øverst i listen med dagens dato |
| Registrere ressurs med posisjon | Fyll ut `/Ressurs/Registrer` med adresse, «Finn på kartet», bekreft plasseringen, legg ved bilde og trykk «Registrer» | Ressursen vises på `/Kart` på riktig sted | ✅ Grønn markør ved riktig adresse. Boblen viser adresse, tilgangsbeskrivelse, kontaktinfo og bildet |

### Docker

| Scenario | Steg | Forventet resultat | Faktisk resultat |
|---|---|---|---|
| Starte appen i Docker | Kjør kommandoene under [Kom i gang](../README.md#kom-i-gang) og åpne http://localhost:8080 | Appen starter, og forsiden vises | ✅ Appen starter, og alle testene over ble kjørt i Docker |

</details>
