# Library APIs – REST, SOAP, GraphQL og gRPC

Fire API'er til samme bibliotek, skrevet i ASP.NET Core 8 med EF Core og SQLite.
Alle fire bruger samme `Library.Data` (entiteter + `LibraryContext`) og samme `library.db`.

## Struktur

```
LibraryApi/
├── Library.sln
├── library.db                ← den udleverede SQLite-database (ikke migreret, så intet databasescript)
├── Library.Data/             ← entiteter + LibraryContext (deles af alle fire API'er)
├── Library.Rest/             ← REST API       → http://localhost:5000      (Swagger: /swagger)
├── Library.Soap1/            ← SOAP API       → http://localhost:5242/soap (WSDL: /soap?wsdl)
├── Library.GraphQL/          ← GraphQL API    → http://localhost:5172/graphql
├── Library.Grpc/             ← gRPC API       → localhost:5003 (HTTP/2)
└── postman/                  ← Postman-collection + environment (REST, SOAP, GraphQL)
```

Portene er forskellige, så alle fire kan køre samtidig.

## Kom i gang i Rider

1. Åbn `Library.sln` i Rider. NuGet-pakkerne hentes automatisk ved første build.
2. `library.db` ligger ved siden af `Library.sln`, og alle projekter finder den selv (`AddLibraryDb` i `Library.Data`).
3. Vil du kigge i databasen: **Database**-vinduet → **+ → Data Source → SQLite** → vælg `library.db`.
   Mappingen til tabellerne `tauthor`, `tpublishingcompany` og `tbook` står i `Library.Data/LibraryContext.cs`.
4. Vælg run-konfigurationen for det API, du vil køre (fx **Library.Rest: http**), og tryk Run.
   Alle på én gang: højreklik på solution → **Run Multiple Projects** / lav en *Compound* run-konfiguration.

## Test i Postman

1. **Import** → vælg begge filer i `postman/`:
   - `Library.postman_collection.json` (mapper: REST, SOAP, GraphQL)
   - `Library.postman_environment.json` (environment **Library - local** med `restUrl`, `soapUrl`, `graphqlUrl`, `grpcUrl`)
2. Vælg environment **Library - local** øverst til højre.
3. Start det API, du vil teste, og kør dets mappe med **Run folder** (Collection Runner).
   Hver mappe opretter sine egne testdata, tester alle operationer inkl. fejl og rydder op bagefter.
4. **gRPC** kan ikke gemmes i en eksporteret Postman-collection. Brug **New → gRPC** → `localhost:5003` →
   **Use server reflection** (eller importér `Library.Grpc/Protos/library.proto`). Se gRPC-afsnittet nedenfor.

Alternativt har REST og gRPC en `requests.http` til Riders HTTP Client.

> Tip: databasen ændres, når man tester. Kør `git checkout library.db` for at få den originale tilbage, fx før aflevering.

---

# REST API (`Library.Rest`)

## Endpoints

| Ressource | Metode | Route | Svar |
|---|---|---|---|
| Books | GET | `/api/books?limit=10&offset=0` | 200 `{ items, total, limit, offset }` |
| | GET | `/api/books/{id}` | 200 / 404 |
| | POST | `/api/books` | 201 + `Location` / 400 |
| | PUT | `/api/books/{id}` | 200 / 400 / 404 |
| | DELETE | `/api/books/{id}` | 204 / 404 |
| Authors | GET | `/api/authors` | 200 |
| | GET | `/api/authors/{id}` | 200 / 404 |
| | POST | `/api/authors` | 201 / 400 |
| | PUT | `/api/authors/{id}` | 200 / 400 / 404 |
| | DELETE | `/api/authors/{id}` | 204 / 404 / 409 |
| Publishers | GET | `/api/publishers` | 200 |
| | GET | `/api/publishers/{id}` | 200 / 404 |
| | POST | `/api/publishers` | 201 / 400 |
| | PUT | `/api/publishers/{id}` | 200 / 400 / 404 |
| | DELETE | `/api/publishers/{id}` | 204 / 404 / 409 |

`limit` skal være 1–100 (standard 10), og `offset` skal være ≥ 0 (standard 0).

## Body-eksempler

```json
// POST/PUT /api/books
{ "title": "Den afrikanske farm", "authorId": 1, "publisherId": 1, "publishingYear": 1937 }

// POST/PUT /api/authors
{ "name": "Karen", "surname": "Blixen" }

// POST/PUT /api/publishers
{ "name": "Gyldendal" }
```

## Fejlhåndtering

Alle fejl returneres som `application/problem+json` (RFC 7807):

- **400** – manglende eller ugyldige felter (efternavn må være tomt, fx "Aristotle"), `publishingYear` < 1900, eller `authorId`/`publisherId`, der ikke findes
- **404** – id'et findes ikke
- **409** – sletning af en forfatter eller et forlag, som en bog refererer til

---

# SOAP API (`Library.Soap1`)

SOAP 1.1 (document/literal) i ASP.NET Core 8, uden ekstra NuGet-pakker: beskederne håndteres med .NET's indbyggede `XmlSerializer` og LINQ to XML.

| | |
|---|---|
| Endpoint | `POST http://localhost:5242/soap` (header `Content-Type: text/xml; charset=utf-8`) |
| WSDL | `GET http://localhost:5242/soap?wsdl` |

## Filer

| Fil | Indhold |
|---|---|
| `library.wsdl` | Kontrakten: XSD-typerne `Book`, `Author`, `PublishingCompany`, request/response-elementer, de tre faults og binding |
| `Contracts/Messages.cs` | C#-klasser, der svarer til XSD'en (`[XmlElement("...")]` = navnet i XML'en) |
| `LibrarySoapService.cs` | De 14 operationer med EF Core og validering – ingen XML her |
| `SoapEndpoint.cs` | Læser `<soap:Envelope>`, finder operationen, XML ↔ C#, laver `<soap:Fault>` |
| `SoapFaultException.cs` | `NotFoundFault`, `ValidationFault`, `ConflictFault` |

## Sådan virker et kald

1. Klienten sender `POST /soap` med en `<soap:Envelope>`. Første element i `<soap:Body>` er fx `<lib:GetBookByIdRequest>`.
2. `SoapEndpoint` fjerner `Request` fra navnet → operationen `GetBookById` → slår request-typen op i `Operations`.
3. `XmlSerializer` laver XML'en om til et C#-objekt, og `LibrarySoapService.GetBookById` kaldes.
4. Resultatet serialiseres som `<GetBookByIdResponse>` i en ny envelope (HTTP 200).
5. Kaster servicen en `SoapFaultException`, svares der i stedet med en `<soap:Fault>` (HTTP 500, som SOAP 1.1 kræver):

```xml
<soap:Fault>
  <faultcode>soap:Client</faultcode>
  <faultstring>Author 1 cannot be deleted: referenced by 1 book(s)</faultstring>
  <detail>
    <tns:ConflictFault xmlns:tns="http://library.ek.dk/soap">
      <tns:message>Author 1 cannot be deleted: referenced by 1 book(s)</tns:message>
    </tns:ConflictFault>
  </detail>
</soap:Fault>
```

## Eksempel-request

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:lib="http://library.ek.dk/soap">
  <soapenv:Body>
    <lib:GetBookByIdRequest>
      <lib:id>1000</lib:id>
    </lib:GetBookByIdRequest>
  </soapenv:Body>
</soapenv:Envelope>
```

## Operationer

| Operation | Input | Output | Faults |
|---|---|---|---|
| CreateBook | title, authorId, publishingCompanyId, publishingYear | id | ValidationFault |
| GetBookById | id | Book | NotFoundFault |
| UpdateBook | id + alle felter | Acknowledgement | ValidationFault, NotFoundFault |
| DeleteBook | id | Acknowledgement | NotFoundFault |
| CreateAuthor | name, surname | id | ValidationFault |
| GetAuthorById | id | Author | NotFoundFault |
| ListAuthors | – | Author[] | – |
| UpdateAuthor | id, name, surname | Acknowledgement | ValidationFault, NotFoundFault |
| DeleteAuthor | id | Acknowledgement | NotFoundFault, ConflictFault |
| CreatePublishingCompany | name | id | ValidationFault |
| GetPublishingCompanyById | id | PublishingCompany | NotFoundFault |
| ListPublishingCompanies | – | PublishingCompany[] | – |
| UpdatePublishingCompany | id, name | Acknowledgement | ValidationFault, NotFoundFault |
| DeletePublishingCompany | id | Acknowledgement | NotFoundFault, ConflictFault |

`Acknowledgement` = `<success>true</success><message>Book 1000 updated</message>`.

**ValidationFault:** manglende/tomme felter, forkert type (fx `<id>abc</id>`), `publishingYear` < 1900 eller i fremtiden,
og `authorId`/`publishingCompanyId`, der ikke findes. Efternavn skal være med, men må være tomt (`<surname/>`).
Mangler `id` helt (fx i GetBookById), svares der også med ValidationFault, da det er en ugyldig request.

---

# GraphQL API (`Library.GraphQL`)

ASP.NET Core 8 + HotChocolate. Endpoint: `POST http://localhost:5172/graphql`
(åbn adressen i browseren for HotChocolates indbyggede editor, *Nitro*).

## Skema

| | Felter / operationer |
|---|---|
| Typer | `Book` (id, title, publishingYear, authorId, author, publisherId, publisher), `Author` (id, name, surname, books), `Publisher` (id, name, books) |
| Query | `bookById(id)`, `authors`, `publishers` |
| Mutation | `createBook`, `updateBook`, `deleteBook`, `createAuthor`, `updateAuthor`, `deleteAuthor`, `createPublisher`, `updatePublisher`, `deletePublisher` |

`Query.cs` og `Mutation.cs` er almindelige C#-klasser; HotChocolate laver skemaet ud fra metoderne (`GetBookById` → `bookById`).

## Eksempler

```graphql
# Én bog med forfatter og forlag i ét kald
query {
  bookById(id: 1000) {
    title
    publishingYear
    author { name surname }
    publisher { name }
  }
}

# Forfattere med deres bøger
query {
  authors { name surname books { title } }
}

mutation {
  createBook(title: "Refactoring", authorId: 1, publisherId: 1, publishingYear: 1999) { id title }
}
```

## Fejlhåndtering

Fejl kommer i `errors` med en kode i `extensions.code`:

- **NOT_FOUND** – id'et findes ikke (`bookById` med et ukendt id giver i stedet `null`)
- **VALIDATION_ERROR** – tom titel/navn, `publishingYear` < 1900, eller forfatter/forlag, der ikke findes
- **CONFLICT** – sletning af en forfatter eller et forlag, som en bog refererer til

---

# gRPC API (`Library.Grpc`)

ASP.NET Core 8 + `Grpc.AspNetCore`. Bruger samme `Library.Data` og `library.db` som REST.

## Kør

1. Vælg run-konfigurationen **Library.Grpc: http** og tryk Run. Serveren lytter på `localhost:5003` (HTTP/2 uden TLS).
2. Åbn `Library.Grpc/requests.http` og klik ▶ ud for en request.

**Postman:** New → **gRPC** → URL `localhost:5003` → vælg **Use server reflection** (eller importér `Protos/library.proto`) → vælg metode → skriv beskeden som JSON → Invoke.

## Filer

| Fil | Indhold |
|---|---|
| `Protos/library.proto` | Kontrakten: service + beskeder. C#-koden genereres ud fra den ved build. |
| `Services/LibraryGrpcService.cs` | De tre RPC'er + validering |
| `Services/BookBroadcaster.cs` | Holder styr på WatchBooks-abonnenter (én `Channel` pr. klient) |
| `Program.cs` | Registrerer servicen, server reflection og broadcasteren (singleton) |

## RPC'er

| RPC | Type | Request | Svar |
|---|---|---|---|
| `GetBookById` | Unary | `{ id }` | `Book` |
| `CreateBook` | Unary | `{ title, author_id, publisher_id, publishing_year }` | `{ id, book }` |
| `WatchBooks` | Server streaming | `{}` | `stream Book` |

`Book` = `{ id, title, author_id, publisher_id, publishing_year }`

## Sådan virker WatchBooks

1. En klient kalder `WatchBooks`. Serveren opretter en kø (`Channel<Book>`) til den i `BookBroadcaster` og holder streamen åben.
2. En anden klient kalder `CreateBook`. Bogen gemmes, og `BookBroadcaster.Publish` lægger den i alle køerne.
3. Hver `WatchBooks`-stream læser fra sin kø og sender bogen til sin klient.
4. Lukker klienten streamen, udløses `CancellationToken`, og abonnementet fjernes.

**Demo:** Start `WatchBooks` i Postman, og kør `CreateBook` fra Rider (eller en anden Postman-fane). Bogen dukker op i Postman med det samme.

Kun bøger oprettet via gRPC sendes ud. Bøger oprettet via de andre API'er kører i en anden proces og kommer ikke med.

## Fejlhåndtering

Fejl returneres som gRPC-statuskoder:

- **NOT_FOUND** – bogen findes ikke
- **INVALID_ARGUMENT** – tom titel, `publishing_year` < 1900, eller `author_id`/`publisher_id`, der ikke findes (alle fejl i én besked)

Valideringsreglerne er de samme som i REST-API'et.

---

# Fordele og ulemper i dette scenarie

| | Fordele | Ulemper |
|---|---|---|
| **REST** | Simpelt og velkendt; CRUD på tre ressourcer passer 1:1 med GET/POST/PUT/DELETE; HTTP-statuskoder (400/404/409) og caching af GET gratis; pagination er naturligt med `?limit=&offset=`; kan bruges fra alle klienter, også browseren. | Ingen streng kontrakt (kun hvis man tilføjer OpenAPI/Swagger); en bog med forfatter- og forlagsnavn kræver 3 kald (*under-fetching*), og man får altid alle felter (*over-fetching*). |
| **SOAP** | Streng, maskinlæsbar kontrakt (WSDL/XSD) med typer, regler (`minInclusive 1900`) og typede faults; klienter kan genereres automatisk; godt til integration med ældre enterprise-/offentlige systemer. | Meget verbost XML; alt er `POST` til én URL, så ingen HTTP-caching og statuskoderne siger ikke meget (fejl = 500); WSDL og C#-klasser skal holdes ens i hånden (contract-first); overkill til et lille bibliotekskatalog. |
| **GraphQL** | Klienten vælger præcis de felter, den skal bruge, og kan hente en bog med forfatter og forlag i ét kald; stærkt typet, selvdokumenterende skema; ét endpoint. | Næsten altid HTTP 200 (fejl ligger i `errors`), så HTTP-caching og statuskoder kan ikke bruges; risiko for tunge, dybt indlejrede queries (fx `authors { books { author { books … } } }`); mere kompleks server. |
| **gRPC** | Hurtig binær Protobuf over HTTP/2; genererede, typede klienter ud fra `.proto`; **streaming** (`WatchBooks`) giver besked med det samme, når en bog oprettes – det kræver polling eller ekstra teknologi i de andre tre. | Kan ikke kaldes direkte fra en browser (kræver gRPC-Web/proxy); beskederne er ikke læsbare for mennesker; kræver specialværktøjer (Postman gRPC, grpcurl). |

**Konklusion:** Til et offentligt bibliotekskatalog er REST det mest praktiske valg, og GraphQL er stærkt, hvis en frontend skal vise bøger sammen med forfattere og forlag. gRPC passer bedst til intern kommunikation mellem services og til live-opdateringer som "nye bøger". SOAP giver mest mening, hvis biblioteket skal integreres med eksisterende systemer, der kræver en formel WSDL-kontrakt.
