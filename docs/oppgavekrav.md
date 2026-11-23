# Oppgavekrav

[← Tilbake til README](../README.md)

Her ligger kravene fra Oppgave 1 og 2 i IS-200, hvor i koden de er løst, og hva vi endret etter tilbakemelding. Siden har også den fulle lista over sider og endepunkter, arbeidsflyten vår og kjente begrensninger. Alt dette sto tidligere i README-en.

## Krav i Oppgave 1

| # | Krav | Hvor | Status |
|---|---|---|---|
| – | Kjøres i Docker | `Dockerfile`, [docs/drift.md](drift.md) | Ferdig |
| 1 | Controller, view-modell og view | `Controllers/`, `Models/ViewModeller/`, `Views/` | Ferdig |
| 2 | Responsive sider med dynamisk innhold fra webserver | Razor-views, `site.css`, kartdata som JSON | Ferdig |
| 3 | Håndterer GET og POST | Se [Sider og endepunkter](#sider-og-endepunkter) | Ferdig |
| 4 | Skjema, og data vises på en annen side | Behov: `/Behov/Registrer` → `/Behov` og `/Oversikt` | Ferdig |
| 5 | Kart, og data fra kartet vises på en annen side | Posisjonsvelger i skjemaene → punkter på `/Kart` | Ferdig |
| 6 | Dokumentasjon om drift, arkitektur og testing | [docs/drift.md](drift.md), [docs/arkitektur.md](arkitektur.md), [docs/testing.md](testing.md) | Ferdig |
| 7 | Dokumentasjon i koden | Norske kommentarer og XML-summaries i koden | Ferdig |
| 8 | Bruk av KI | [docs/ki-bruk.md](ki-bruk.md) | Ferdig |

## Krav i Oppgave 2

| # | Krav | Hvor i koden / dokumentasjonen | Status |
|---|---|---|---|
| 1 | Kobling mot MariaDB, app og database i Docker | `docker-compose.yml`, `Data/BeredskapsDbContext.cs`, `Migrations/`, `Services/Ef*Repository.cs`, [docs/drift.md](drift.md) | Ferdig |
| 2 | Registrering av ny bruker, lagret i databasen | `Controllers/KontoController.cs` (`Registrer`, og `EndreProfil`/`EndrePassord` på Min side), `Models/Bruker.cs` (tabellen `AspNetUsers`) | Ferdig |
| 3 | Autentisering, autorisering, og legge til, endre og slette ressurs | `KontoController`, `[Authorize(Roles)]` på controllerne, `RessursController` (`Registrer`, `Rediger`, `TrekkTilbake`) | Ferdig |
| 4 | Funksjoner som beskrevet av oppdragsgiveren | [Funksjoner](../README.md#funksjoner), `Services/TildelingTjeneste.cs`, [docs/arkitektur.md](arkitektur.md) avsnitt 1 og 5b | Ferdig |
| 5 | Kartet i skjemaet setter et punkt | `wwwroot/js/posisjonsvelger.js`, `Views/Behov/Registrer.cshtml`, `Views/Ressurs/Registrer.cshtml` | Ferdig |
| 6 | Mobiltilpasset frontend | `wwwroot/css/site.css` (`@media`), `_Layout.cshtml`, [mobiltest](testing.md#mobiltest) | Ferdig |
| 7 | Enkel oversikt over ressurser | `/Oversikt`, `/Ressurs` (liste med filter), lista under kartet, `/Ressurs/Detaljer`, `/Tildeling` | Ferdig |
| 8 | Sikkerhetstiltak | `Program.cs`, `Sikkerhet/SikkerhetsheadereMiddleware.cs`, [docs/arkitektur.md](arkitektur.md) avsnitt 6 | Ferdig |
| 9 | Testing (enhet, system, sikkerhet, brukervennlighet) | `Beredskapsportal.Tests/` (162 tester), [Testing](testing.md) | Enhet, system, sikkerhet og mobil ferdig. Brukervennlighetstest delvis (privat aktør testet 12.11.2026) |
| 10 | Dokumentasjon om drift, arkitektur og testing | [docs/drift.md](drift.md), [docs/arkitektur.md](arkitektur.md), [Testing](testing.md) | Ferdig |
| 11 | Dokumentasjon i koden | `/// <summary>` på alle klasser, norske kommentarer i C#, Razor (`@* *@`) og JavaScript | Ferdig |
| 12 | Bruk av KI | [docs/ki-bruk.md](ki-bruk.md) | Ferdig |

### Endret etter tilbakemelding på innlevering 1

| Tilbakemelding | Hva vi har gjort |
|---|---|
| Legg inn konkrete prompter i KI-delen | Lagt til flere prompter vi faktisk har brukt, se [docs/ki-bruk.md](ki-bruk.md) |
| Mangler automatiske tester for POST-håndtering av behov og ressurser | Nye tester i `BehovControllerTester.cs` og `RessursControllerTester.cs`: gyldig skjema lagres, ugyldig skjema lagres ikke, manglende plassering, for mange og for store bilder |
| Mangler tester for validering av skjemaer | Ny `ValideringTester.cs` med de ekte valideringsreglene for behov, ressurs, innlogging og registrering (påkrevde felt, telefon, e-post, maks lengde) |
| Mangler tester for `/Kart/Behov` og `/Kart/Ressurser` | Ny `KartControllerTester.cs`: riktige felt i JSON-svaret, bare godkjente ressurser, tildelte ressurser merket, bare GET og bare offentlige aktører |
| Rate limiting, f.eks. ved innlogging | Maks 10 forsøk per minutt per IP på innlogging og registrering (`Sikkerhet/Fartsgrenser.cs`), i tillegg til utestenging etter 5 feil passord |
| SQL-injeksjon | All databasetilgang går gjennom EF Core med parametere, ingen egen SQL. Testet manuelt, se [sikkerhetstesten](testing.md#sikkerhetstest) |
| Beskyttelse mot XSS | Razor-encoding, ingen `@Html.Raw`, `trygg()` i JavaScript og Content-Security-Policy uten `unsafe-inline`. Testet manuelt, se [sikkerhetstesten](testing.md#sikkerhetstest) |
| Kontroll av opplastede filer | Innholdet i filen sjekkes nå, ikke bare endelsen (`Sikkerhet/Bildesjekk.cs`), og hele skjemaet har en grense på 26 MB. Tester i `SikkerhetTester.cs` |

I tillegg fant vi at tekstfeltene i skjemaene ikke hadde maks lengde, mens kolonnene i databasen har det. For lang tekst ga en feil fra databasen. Nå har feltene `[StringLength]` med samme lengde, og brukeren får en melding.

## Sider og endepunkter

| Metode | Adresse | Beskrivelse | Tilgang |
|---|---|---|---|
| GET | `/` | Forside | Alle |
| GET / POST | `/Konto/Registrer` | Registrere bruker (offentlig eller privat aktør) | Alle |
| GET / POST | `/Konto/LoggInn` | Logge inn | Alle |
| POST | `/Konto/LoggUt` | Logge ut | Innlogget |
| GET | `/Konto/Profil` | Min side | Innlogget |
| GET / POST | `/Konto/EndreProfil` | Endre navn og e-post (e-post krever nåværende passord) | Innlogget |
| GET / POST | `/Konto/EndrePassord` | Endre passord | Innlogget |
| GET | `/Oversikt` | Dashbord | Innlogget |
| GET | `/HvaTrengs` | Hva som trengs, uten hvem som har meldt | Innlogget |
| GET | `/Behov` | Alle behov med filter | Offentlig aktør |
| GET / POST | `/Behov/Registrer` | Melde behov | Offentlig aktør |
| GET | `/Behov/Detaljer/{id}` | Detaljside med historikk | Offentlig aktør |
| GET / POST | `/Behov/Rediger/{id}` | Endre behov | Den som meldte det, og admin |
| GET / POST | `/Behov/TrekkTilbake/{id}` | Trekke tilbake eget behov | Den som meldte det, og admin |
| GET | `/Ressurs` | Liste over ressurser med filter på type og status, sortert med tilgjengelige øverst og avviste nederst. Private aktører («Mine ressurser») ser bare type, adresse og status, og en knapp for å endre | Innlogget. Offentlige ser godkjente og egne, private bare egne, admin alle |
| GET / POST | `/Ressurs/Registrer` | Tilby ressurs med bilder | Innlogget |
| GET | `/Ressurs/Detaljer/{id}` | Detaljside med bilder, kart og historikk | Eier, admin, eller offentlig aktør når godkjent |
| GET / POST | `/Ressurs/Rediger/{id}` | Endre ressurs | Eier og admin |
| GET / POST | `/Ressurs/TrekkTilbake/{id}` | Trekke tilbake (slette) ressurs | Eier og admin |
| GET | `/Ressurs/Bilde/{id}` | Bilde av ressurs | Samme som detaljsiden |
| GET | `/Kart` | Kart med liste og filter | Offentlig aktør |
| GET | `/Kart/Behov`, `/Kart/Ressurser` | Punkter som JSON til kartet | Offentlig aktør |
| GET | `/Foresporsel/FinnRessurs/{id}` | Forslag til ressurser for eget behov | Offentlig aktør |
| POST | `/Foresporsel/Foresporr` | Forespørre en ressurs | Offentlig aktør |
| GET | `/Tildeling` | Alle tildelinger med filter | Offentlig aktør |
| GET | `/Admin` | Godkjenning av ressurser, brukere og forespørsler | Admin |
| GET / POST | `/Admin/Tildel/{id}` | Tildele ressurser til et behov | Admin |
| POST | `/Admin/GodkjennRessurs`, `AvvisRessurs`, `GodkjennBruker`, `AvvisBruker`, `GodkjennForesporsel`, `AvslaForesporsel`, `FjernTildeling` | Handlinger på adminsiden | Admin |

**Flyt i et skjema (eksempel):** `GET /Behov/Registrer` viser skjemaet → brukeren sender det inn med `POST` → controlleren validerer og lagrer i MariaDB via repository → brukeren sendes videre til `/Behov`, der det nye behovet vises.

## Arbeidsflyt

- Én branch per oppgave: `feature/kort-beskrivelse` eller `fix/kort-beskrivelse`
- All kode inn via pull request, godkjent av andre gruppemedlemmer
- Branches slettes etter merge
- Oppgaver følges opp i Trello: Backlogg → Klar til start → Pågår → Til review → Ferdig

## Kjente begrensninger

- Appen kjører over http, ikke https, i Docker. Den lytter bare på `127.0.0.1`, så den er ikke tilgjengelig fra andre maskiner.
- Hver utvikler har sin egen lokale database. Det finnes ingen felles server.
- Det er ikke satt opp automatisk backup av databasen. Data ligger i Docker-volumet `dbdata`.
- Kartfliser og adressesøk hentes fra Kartverket, så kartet krever internett.
- Se [docs/drift.md](drift.md) for detaljer.
