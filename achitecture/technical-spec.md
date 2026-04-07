# 🧩 Teknisk Specifikation – Modulär Idrottsplattform

**Version:** 1.0  
**Datum:** 2026‑02‑26  
**Status:** Fastställd arkitektur och datamodell  
**Omfattar:** Auth, Time, Results, Competition, Gateway, PWA:er, Main UI, DB:er, CI/CD, säkerhet

***

# 1. Systemöversikt

Systemet består av en **mikrotjänstarkitektur** med fyra backends, tre separata frontendprojekt och en enda exponerad gateway. Tjänsterna är frikopplade och utbyter ingen databasinformation direkt — all kommunikation sker över HTTP via Gateway‑API eller via interna API‑nycklar.

**Backendtjänster:**

| Tjänst                  | Ansvar                                                                            |
| ----------------------- | --------------------------------------------------------------------------------- |
| **Auth-service**        | Identitet, roller, permissions, login med magic‑link/OTP/TOTP, JWT‑tokenhantering |
| **Time-service**        | Tidsrapportering, attest, reseräkning, traktamente & milersättning                |
| **Group-service**       | Grupphantering, grupper, gruppmedlemmar och ledare                                |
| **Results-service**     | Resultat, atleter, versionerad namn-/licenshistorik, snapshots                    |
| **Competition-service** | Tävlingar, grenar och anmälningar                                                 |

***

# 2. Backend‑arkitektur – Mikrotjänster (Action‑baserad Slim‑modell)

Detta kapitel beskriver backend‑arkitekturen för samtliga mikrotjänster, byggd på Slim 4 med **single‑action invokable handlers**, en tydlig domänmodell och strikt avskiljda tjänster/datalager.

***

## 2.1 Gemensamma principer

### **1) En tjänst = en databas**

Varje mikrotjänst äger sin egen databas och sin egen domän:

*   `auth_db`
*   `time_db`
*   `gourp_db`
*   `results_db`
*   `competition_db`

Ingen tjänst får direkt läsa en annan tjänsts databas.  
Allt utbyte sker via Gateway‑API.

***

### **2) Endast Gateway‑API är exponerad externt**

*   Gateway är den *enda* publika HTTP‑ingången.
*   Mikrotjänster körs på ett privat Docker‑nät (`internal` network).
*   Gateway ansvarar för:
    *   JWT‑validering
    *   klientpolicy (via `client_type`)
    *   routing
    *   CORS
    *   rate limiting
    *   hälsokontroller

***

### 3) Intern kommunikation sker via HTTP + shared secret

Tjänster kommunicerar:

*   via interna URL:er, t.ex. `http://results-service.internal/…`
*   med en `X-Service-Key`‑header
*   utan att exponeras offentligt

Det gör systemet mycket robust och lätt att skala.

***

### 4) Action‑baserad kodstruktur 

Alla tjänster använder **Slims action‑mönster**:

✔ **En route = en Action-klass** (`*Action.php`)  
✔ Varje action är en **invokable handler**  
✔ Actions ska vara tunna (I/O, validering, anropa service)  
✔ All logik ligger i services och domain‑lager

### Uppdaterad katalogstruktur per tjänst:

    /src
      /Application
        /Actions          # Invokable handlers 
        /Services         # Affärslogik
        /Validators
      /Domain
        /Entities         # Domänobjekt
        /ValueObjects
        /Repositories     # Interface för datalager
      /Infrastructure
        /Persistence      # DBAL-implementationer av Repositories
        /Http             # Middlewares, helpers
    /config
      di.php              # PHP-DI definitions
      routes.php          # Route → Action-binding
    /migrations
    /tests

#### Exempel på en typisk action:

    CreateTimeEntryAction
    ApproveTravelReportAction
    RecordResultAction
    ImportResultsAction
    RegisterEntryAction
    RequestMagicLinkAction
    ConsumeMagicLinkAction

***

### 5) Tjänster är helt frikopplade

Varje tjänst:

*   har sin egen kodbas
*   deployas separat
*   versioneras separat
*   har egna migrations (Phinx)
*   har sin egen CI/CD pipeline

Det gör arkitekturen robust och flexibel.

***

### 6) Kommunikation är synkron via HTTP (REST)

En senare expansion med:

*   async events (Kafka/RabbitMQ)
*   materialiserade vyer
*   cache‑lager

…är möjlig tack vare ren separation.

***

## 2.2 Teknikstack per tjänst

Alla backendtjänster använder samma grundstack för enhetlighet:

### Slim Framework 4

*   PSR‑7 / PSR‑15 kompatibel
*   perfekt för action‑baserat API
*   mycket snabb och resurssnål

### PHP‑DI

*   Fullständig beroendeinjektion
*   Gör Actions testbara och lätta att använda
*   Inga service locators

### Doctrine DBAL

*   Lättviktig DB‑access
*   Utmärkt för mikrotjänster där ren SQL behövs
*   Undviker overhead från full ORM

### Phinx (migreringar)

*   Egen `phinx.php` per tjänst
*   Versionerade migrations
*   Idempotenta seeds där nödvändigt (t.ex. roller i Auth)

### Monolog

*   Standardiserad loggning
*   JSON‑format för gateway/log pipeline
*   Logg per tjänst

### JWT (firebase/php-jwt)

*   Access/refresh‑token
*   `client_type`‑claim
*   Signering via HS256 eller ES256

***

## 2.3 Testförfarande och kvalitetsverktyg

Detta avsnitt beskriver hur testning, statisk analys, kodstil och mutationstester hanteras i backend‑utvecklingen. Målet är att säkerställa hög kodkvalitet, stabil funktionalitet och en modern utvecklingsprocess — utan att införa onödigt komplexa verktyg. Alla verktyg ska kunna köras lokalt och inkluderas automatiskt i CI/CD‑flödet.

***

### 1. Enhetstester (PHPUnit)

Varje mikrotjänst innehåller ett `tests/`‑träd med:

*   **enhetstester** för Services och Domain‑objekt
*   **integrationstester** för Actions och Repositories
*   tester som körs både lokalt och via CI

Körning lokalt:

```bash
vendor/bin/phpunit
```

I CI körs samma tester med **code coverage** aktiverat.

***

### 2. Statisk analys (PHPStan)

PHPStan används för att tidigt upptäcka:

*   typfel
*   ogiltiga anrop
*   felaktiga signaturer
*   odefinierade variabler
*   potentiella buggar i affärslogik och domänlager

Varje tjänst har sin egen `phpstan.neon` med en rekommenderad nivå på **level max**.

```bash
vendor/bin/phpstan analyse src --level=max
```

***

### 3. Kodstandard (PHPCS)

PHPCS säkerställer:

*   konsekvent kodstil
*   att projektets egna kodregler följs
*   PSR‑12 som grundstandard
*   utökningar för interna preferenser

Körning:

```bash
vendor/bin/phpcs src
```

***

### 4. Automatisk formattering (PHP‑CS‑Fixer / PHPCBF)

*   **PHP‑CS‑Fixer** används för avancerad, automatiserad formatering enligt projektets egna regler.
*   **PHPCBF** (som medföljer PHPCS) kan användas för att autofixa kompatibla regelbrott.

Exempel:

```bash
vendor/bin/php-cs-fixer fix
```

eller:

```bash
vendor/bin/phpcbf src
```

***

### 5. Mutationstester (Infection)

Infection säkerställer att tester inte bara ”täcker” kod, utan också **upptäcker verkliga förändringar i beteende**. Det är ett centralt verktyg för att förbättra testernas kvalitet över tid.

```bash
vendor/bin/infection
```

Mutationstester körs lokalt vid behov och periodiskt i CI (pga längre körtider).

***

### 6. Automatisk modernisering av kod (Rector)

Rector används för att:

*   modernisera syntaktiska konstruktioner
*   reducera teknisk skuld
*   implementera kodregler över hela koden utan handpåläggning
*   uppgradera PHP‑språkfunktioner när projektet växer

Körning:

```bash
vendor/bin/rector process src
```

Rector körs som **dry‑run** i CI för att upptäcka förbättringsbehov.

***

### 7. Kvalitetskontroll vid commit (GrumPHP)

GrumPHP konfigureras som en del av repository‑initiering och kör automatiskt:

*   PHPStan
*   PHPCS
*   PHP‑CS‑Fixer (dry‑run)
*   PHPUnit (snabba tester)
*   Rector (dry‑run)

Ingen commit tillåts om någon kontroll fallerar.

Initiering:

```bash
vendor/bin/grumphp git:init
```

Detta gör att utvecklare får snabb, lokal återkoppling innan kod pushas.

***

### 8. Code coverage

Coverage mäts med PHPUnit och ska vara:

*   **hög** för Services och Domain
*   **rimlig** för Actions
*   **tillräcklig** för Repositories (via integrationstester)

Code coverage används inte som strikt krav, men som verktyg för att bedöma testdjup.

***

### 9. Samma verktyg körs lokalt och i CI/CD

Det säkerställer att kvaliteten är konsekvent.

#### Lokalt (via kommandon eller GrumPHP):

1.  PHPCS
2.  PHP‑CS‑Fixer (dry‑run)
3.  PHPStan
4.  PHPUnit (+ coverage vid behov)
5.  Infection
6.  Rector (dry-run)
7.  GrumPHP vid commit

#### CI/CD pipeline kör i ordning:

1.  Installera beroenden
2.  PHPCS
3.  PHPStan
4.  PHP‑CS‑Fixer (dry‑run)
5.  Rector (dry-run)
6.  PHPUnit (med coverage‑rapport)
7.  Infection (full körning)
8.  Bygg och publicera Docker‑image
9.  Deploy + kör Phinx‑migreringar

***

### 10. Sammanfattning av teststrategin

Det valda paketet ger en **balanserad, lärorik och professionell** testmiljö:

*   PHPUnit → korrekt funktion
*   PHPStan → robust typ- och logikanalys
*   PHPCS / PHP‑CS‑Fixer → snygg och konsekvent kod
*   Infection → hög testkvalitet
*   Rector → långsiktig modernisering
*   GrumPHP → snabb lokal kvalitetssäkring

Detta är en modern teststack som passar perfekt för ett multi‑service‑projekt utan att vara överdrivet komplex.

***

# 3. Frontend-arkitektur

## 3.1 PWA Tränare

*   Plattform: Vue 3 + Vite + TypeScript
*   Funktion: tidsrapportering, resor, kvitton
*   Långa sessioner via Refresh Tokens
*   Offline‑stöd för lokala utkast

## 3.2 PWA Aktiva

*   Funktion: resultatvisning, PB/ÅB
*   Publikt läge → ingen login
*   Privat läge → magic‑link
*   Minimalistisk UI

## 3.3 Main UI

*   Vue 3 + Vite + Vuetify
*   Modulär struktur:
        src/modules/
          time-approval/
          travel-approval/
          results/
          competitions/
          permissions/
*   Rollstyrd navigation

***

# 4. Gateway‑API & Säkerhetsmodell

## 4.1 Grundprinciper

*   **ENDPOINTS:** Endast gateway är exponerad.
*   **UPSTREAM:** Mikrotjänster är ej åtkomliga från internet.
*   **CORS:** Hanteras vid gateway.

## 4.2 Tokenmodell

JWT innehåller:

    {
      "sub": "<user_id>",
      "client": "pwa_trainer | pwa_results | main_ui",
      "iat": <timestamp>,
      "exp": <timestamp>
    }

### Access tokens

*   Kortlivade (15–120 min)

### Refresh tokens

*   Längre livstid i PWA-klienter
*   Kortare i Main UI

### Klienttyper (client\_type)

*   `pwa_trainer`
*   `pwa_results`
*   `main_ui`

Gateway avgör vilka endpoints klienttypen får nå.

## 4.3 Inloggningsmetoder

*   **Magic Link**: Primär login
*   **OTP-kod**: Alternativt
*   **TOTP**: För administratörsroller

***

# 5. Datamodell – Principer

*   Alla tjänster har **separata databaser**.
*   Alla PK är **GUID/UUID**.
*   Endast auth\_db innehåller “systemanvändare”.
*   Inga tjänster delar tabeller.
*   Relationspilar mellan tjänster går alltid via API, aldrig SQL.

***

## 4.4 Gateway‑API – Arkitektur, routing och säkerhet (Traefik + PHP‑gateway)

Gateway‑API är systemets **enda publika ingång** och ansvarar för autentisering, klientpolicy och vidarebefordran av requests till interna mikrotjänster.  
Den består av två separata lager:

1.  **Traefik** – ingress, HTTPS, routing, CORS m.m.
2.  **PHP‑gateway (Slim)** – JWT‑verifiering, klientpolicy, proxying

Detta är en branschstandardiserad modell för lätta mikrotjänstplattformar.

***

### 4.4.1 Syfte

Gatewayen har som uppgift att:

*   utgöra **enda exponerade ytan** mot internet
*   säkerställa **JWT‑verifiering** för samtliga inkommande anrop
*   kontrollera **client\_type** för att styra vilka tjänster en klient får nå
*   forwarda requests vidare till rätt mikrotjänst
*   tillämpa ingress‑policyer (TLS, rate limiting, CORS, headers) via Traefik
*   skicka relevanta identitetsdata vidare till underliggande tjänster

Gatewayen innehåller *ingen* affärslogik.

***

### 4.4.2 Arkitekturöversikt

            Internet
                |
                v
       +------------------+
       |      Traefik     |  <-- offentlig ingress
       +------------------+
                |
                v
       +------------------+
       |   PHP Gateway    |  <-- JWT, client_type, proxy
       |   (Slim-app)     |
       +------------------+
         |         |        |
         v         v        v
      Auth       Time     Group
     Service    Service  Service

All kommunikation med mikrotjänster sker på ett **internt Docker‑nät**.

***

### 4.4.3 Routingmodell (Traefik)

Traefik fungerar som ingress‑controller och tar emot all trafik.  
Routing sker via Docker‑labels och skickas till PHP‑gateway‑containern.

Exempel:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.gateway.rule=Host(`api.example.com`)"
  - "traefik.http.services.gateway.loadbalancer.server.port=8080"
```

PHP‑gateway tar därefter över, extraherar path och forwardar till rätt mikrotjänst, t.ex.:

*   `/auth/*`   → auth‑service
*   `/time/*`   → time‑service
*   `/group/*`  → group‑service
*   `/results/*` och `/competition/*` senare när dessa aktiveras

***

### 4.4.4 PHP‑gateway (Slim) – ansvar

PHP‑gatewayn ansvarar för:

#### Autentisering

*   läser `Authorization: Bearer <JWT>`
*   verifierar signatur (HS256/ES256)
*   kontrollerar `sub`, `iat`, `exp`, `nbf`
*   avkodar roller/permissions om de finns i token

#### Klientpolicy

Styr vilka endpoints som en klient får anropa:

| client\_type  | Tillåtna prefix   |
| ------------- | ----------------- |
| `pwa_trainer` | `/time`, `/group` |
| `pwa_results` | `/results`        |
| `main_ui`     | alla tjänster     |

Försök att nå en otillåten endpoint ger **403 Forbidden**.

#### Identitets‑metadata

Gateway lägger till följande headers när den proxar vidare till mikrotjänster:

    X-User-Id: <uuid>
    X-Client-Type: <client_type>
    X-Roles: role1,role2
    X-Permissions: time.submit,time.approve
    X-Request-Id: <uuid>

#### Proxying

Gateway vidarebefordrar:

*   HTTP‑method (GET/POST/PUT/DELETE/PATCH)
*   URL
*   JSON‑payload
*   headers (utom de som gatewayen ska ersätta)

***

### 4.4.5 Autentisering (JWT)

JWT genereras av Auth‑service och verifieras i gatewayen.  
Verifieringsstegen är:

1.  Verifiera signatur
2.  Kontrollera token‑utgång (`exp`) och giltighet (`nbf`, `iat`)
3.  Extrahera claims:
    *   `sub` (user\_id)
    *   `client_type`
    *   `roles`
    *   `permissions`
4.  Tillämpa klientpolicy
5.  Forwarda requesten vidare

Mikrotjänster behöver **inte** känna till JWT – de läser headers från gatewayen.

***

### 4.4.6 Säkerhet (Traefik)

Traefik ansvarar för all infrastrukturrelaterad säkerhet:

#### HTTPS & certifikat

Automatiskt via Let's Encrypt / ACME.

#### CORS

Konfigureras via Traefik‑middleware.

#### Rate limiting

Per route, IP eller path.

#### Headers

HSTS, referrer‑policy, XSS‑skydd m.m.

#### Load balancing

Om du skalar en tjänst till flera containrar.

Detta gör gateway‑koden ren och lätt.

***

### 4.4.7 Logging och trace‑ID

Gatewayen genererar:

    X-Request-Id
    X-Correlation-Id

Som loggas i:

*   Auth‑service
*   Time‑service
*   Group‑service

Det ger konsekvent loggkorrelation genom hela systemet.

***

### 4.4.8 Ej tillåten logik i gatewayen

För att säkerställa separation av ansvar får gatewayen **inte**:

*   köra affärslogik
*   validera JSON‑payloads
*   slå upp databasinformation
*   transformera data mellan tjänster
*   göra domänspecifika beslut
*   hålla state
*   cacha domäninformation
*   göra cross‑service‑orchestration

Gatewayen ska vara:

> **“Secure, simple, stateless.”**

***

### 4.4.9 Framtida expansion

Arkitekturen tillåter att gatewayen senare utökas med:

#### Push‑notifikationer (Web Push)

Gateway som notifieringsentrypoint:  
`POST /notifications/send`

#### API‑versionering

Prefix: `/api/v1/…` → `/api/v2/…`  
Hantera flera versioner parallellt.

***

## 5.1 Auth DB

AuthDB hanterar endast **identitet**, **autentisering** och **auktorisering**.  
Ingen domänspecifik persondata lagras här.  
AuthDB används av samtliga mikrotjänster via Gateway‑API för att:

*   identifiera användare
*   validera sessionsflöden (magic link / OTP / TOTP)
*   fastställa roller och rättigheter

Alla primärnycklar är **UUID** (CHAR(36)).

***

### users

Representerar en systemanvändare och innehåller endast *identitetsrelaterad* information, inte domändata.

| Fält        | Typ                 | Beskrivning                    |
| ----------- | ------------------- | ------------------------------ |
| id          | CHAR(36) PK         | Användarens UUID               |
| email       | VARCHAR(255) UNIQUE | Unik e‑post för inloggning     |
| first\_name | VARCHAR(100)        | Användarens aktuella förnamn   |
| last\_name  | VARCHAR(100)        | Användarens aktuella efternamn |
| created\_at | DATETIME            | När användaren skapades        |
| updated\_at | DATETIME            | När användaren senast ändrades |

**Index:**

*   UNIQUE(email)
*   INDEX(last\_name) *(valfritt, men framtidssäkert)*

**Designnot:**  
Detta namn är **inte** atletens eller tränarens versionshanterade domännamn — det är endast användarens identitetsnamn för UI, audit och login.

***

### roles

Roller används för att gruppera åtkomsträttigheter.

| Fält        | Typ                 | Beskrivning                        |
| ----------- | ------------------- | ---------------------------------- |
| id          | CHAR(36) PK         | UUID                               |
| name        | VARCHAR(100) UNIQUE | Ex: `admin`, `trainer`, `approver` |
| description | VARCHAR(255) NULL   | Valfri beskrivning                 |

***

### permissions

Permissions är detaljerade rättighetsflagor som roller kopplas till.

| Fält        | Typ                 | Beskrivning                          |
| ----------- | ------------------- | ------------------------------------ |
| id          | CHAR(36) PK         | UUID                                 |
| name        | VARCHAR(150) UNIQUE | Ex: `time.approve`, `results.import` |
| description | VARCHAR(255) NULL   | Beskrivning av rättigheten           |

***

### user\_roles

Kopplingstabell mellan users och roles (many-to-many).

| Fält     | Typ                    |
| -------- | ---------------------- |
| user\_id | CHAR(36) FK → users.id |
| role\_id | CHAR(36) FK → roles.id |

**Primärnyckel:** `(user_id, role_id)`  
**Index:** `INDEX(role_id)`

***

### role\_permissions

Kopplingstabell mellan roles och permissions.

| Fält           | Typ                          |
| -------------- | ---------------------------- |
| role\_id       | CHAR(36) FK → roles.id       |
| permission\_id | CHAR(36) FK → permissions.id |

**Primärnyckel:** `(role_id, permission_id)`  
**Index:** `INDEX(permission_id)`

***

### magic\_links

Används för lösenordslös inloggning.  
Tokens lagras som **hash**, aldrig i klartext.

| Fält         | Typ                                              | Beskrivning                      |
| ------------ | ------------------------------------------------ | -------------------------------- |
| id           | CHAR(36) PK                                      | UUID                             |
| user\_id     | CHAR(36) FK → users.id                           | Vem länken gäller                |
| token\_hash  | CHAR(64)                                         | SHA‑256 hash av engångstoken     |
| client\_type | ENUM('pwa\_trainer', 'pwa\_results', 'main\_ui') | Säkerställer korrekt klientflöde |
| expires\_at  | DATETIME                                         | Utgångstid                       |
| consumed\_at | DATETIME NULL                                    | Om/när den användes              |
| created\_at  | DATETIME                                         | Skapad                           |

**Index:**

*   INDEX(user\_id)
*   INDEX(token\_hash)

***

### one\_time\_codes

Engångskoder som kan användas för högrisksituationer, alternativ login eller bekräftelser.

| Fält         | Typ                    |              |
| ------------ | ---------------------- | ------------ |
| id           | CHAR(36) PK            |              |
| user\_id     | CHAR(36) FK → users.id |              |
| code\_hash   | CHAR(64)               | SHA‑256 hash |
| expires\_at  | DATETIME               |              |
| consumed\_at | DATETIME NULL          |              |
| created\_at  | DATETIME               |              |

**Index:**

*   INDEX(user\_id)
*   INDEX(code\_hash)

***

### totp\_secrets

Lagrar TOTP‑hemligheter för tvåfaktorsautentisering (TOTP). Används främst av administratörer, attestanter och andra riskroller.

| Fält           | Typ                        |                       |
| -------------- | -------------------------- | --------------------- |
| user\_id       | CHAR(36) PK, FK → users.id |                       |
| secret         | VARCHAR(255)               | TOTP Base32-hemlighet |
| created\_at    | DATETIME                   | När aktiverad         |
| last\_used\_at | DATETIME NULL              | Senast validerad      |

**Designnot:**  
Separat tabell minskar riskprofilen och gör det enkelt att aktivera/deaktivera TOTP utan att ändra andra user‑fält.

***

### ✔ Sammanfattning

Denna AuthDB‑modell:

*   följer modern praxis för lösenordslös inloggning
*   skiljer korrekt mellan **identitetsnamn** och **domännamn**
*   är säker, minimal och framtidssäker
*   stödjer roll- och rättighetssystem för gateway och UI
*   är perfekt för mikrotjänstarkitektur där Auth är isolerat från domänen

***
## 5.2 Time DB – Datamodell 

TimeDB äger all **tids‑ och reseersättningsdata** för ledare/assistenter.  
Modellen är optimerad för snabb rapportering i UI, enkel attest, säkra löneunderlag och minsta möjliga koppling till andra domäner.

**Grundprinciper**

*   **En tjänst = en databas** (Time‑service äger TimeDB).
*   **GUID/UUID** används som PK (CHAR(36)) där inget annat anges.
*   **Låst historik**: alla belopp som beräknas (timlön per pass, traktamente, milersättning) **fryses** när posten skapas/uppdateras – ingen retroaktiv omräkning.
*   **Namn på ledare/assistenter hämtas från AuthDB** (`auth.users`). TimeDB lagrar inte namn.
*   **Grupper är en egen domän** (Group‑service). TimeDB lagrar endast en **referens** vid träningstid.

***

### Tabellöversikt

*   `time_users` – verksamhetsspecifik profil för ledare/assistenter (1:1 med `auth.users`)
*   `pay_groups` – ersättningskategorier (ex. pass‑typ)
*   `pay_rates` – **versionerad** timlön per ledare & ersättningskategori
*   `time_entries` – rapporterade arbetspass (med förenklad kontext)
*   `travel_reports` – resor (traktamente, milersättning, övriga kostnader) *utan* dagsrader
*   `travel_allowance_rules` – traktamenteregler per land/år
*   `mileage_rules` – milersättning per år & fordonstyp


***

### `time_users`

| Fält          | Typ                               | Beskrivning               |
| ------------- | --------------------------------- | ------------------------- |
| user\_id      | CHAR(36) PK, FK → `auth.users.id` | Samma UUID som i AuthDB   |
| signum        | VARCHAR(50)                       | Internt id/initialer e.d. |
| bank\_account | VARCHAR(50)                       | Konto för utbetalning     |
| created\_at   | DATETIME                          | Skapad                    |
| updated\_at   | DATETIME                          | Senast uppdaterad         |

**Index:** `PK(user_id)`

***

### `pay_groups`

| Fält        | Typ          | Beskrivning                          |
| ----------- | ------------ | ------------------------------------ |
| id          | CHAR(36) PK  | UUID                                 |
| name        | VARCHAR(255) | Ex. “Simpass kväll”, “Assistentpass” |
| description | TEXT NULL    | Förtydligande                        |
| created\_at | DATETIME     | Skapad                               |

**Index:** `PK(id)`

***

### `pay_rates`  (versionerad timlön)

| Fält           | Typ                                | Beskrivning                   |
| -------------- | ---------------------------------- | ----------------------------- |
| id             | CHAR(36) PK                        | UUID                          |
| time\_user\_id | CHAR(36) FK → `time_users.user_id` | Ledare/assistent              |
| pay\_group\_id | CHAR(36) FK → `pay_groups.id`      | Ersättningskategori           |
| hourly\_rate   | DECIMAL(10,2)                      | Timlön                        |
| valid\_from    | DATE                               | Gäller från                   |
| valid\_to      | DATE NULL                          | Gäller till (NULL = pågående) |
| created\_at    | DATETIME                           | Skapad                        |

**Index:**

*   `INDEX(time_user_id)`
*   `INDEX(pay_group_id)`
*   `INDEX(valid_from, valid_to)`

***

### `time_entries`  

| Fält           | Typ                                                       | Beskrivning                                                                   |
| -------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------- |
| id             | CHAR(36) PK                                               | UUID                                                                          |
| trainer\_id    | CHAR(36) FK → `time_users.user_id`                        | Vem passet avser                                                              |
| date           | DATE                                                      | Datum för passet                                                              |
| minutes        | INT                                                       | Arbetad tid (minuter)                                                         |
| pay\_group\_id | CHAR(36) FK → `pay_groups.id`                             | Vald ersättningskategori                                                      |
| pay\_rate\_id  | CHAR(36) FK → `pay_rates.id`                              | **Låst** rate som gällde för datumet                                          |
| status         | ENUM('draft','submitted','approved','rejected')           | Atteststatus                                                                  |
| description    | TEXT NULL                                                 | Valfri kommentar                                                              |
| work\_type     | ENUM('training\_group','competition','education','other') | Kontekstyp                                                                    |
| work\_ref\_id  | CHAR(36) NULL                                             | **Endast** om `work_type='training_group'` – ID från Group‑service            |
| purpose        | TEXT NULL                                                 | **Obligatoriskt** om `work_type≠'training_group'` (tävling/utbildning/övrigt) |
| exported_at     | DATETIME NULL                                             | När posten har exporterats                                                    |
| export_batch_id | CHAR(36) NULL	                                         | Batch id för export                                                           |
| export_attempts | INT DEFAULT 0                                             | Antal försök till export                                                      |
| export_error    | TEXT NULL                                                 | Orsak till exporten misslyckades                                              |
| created\_at    | DATETIME                                                  | Skapad                                                                        |
| updated\_at    | DATETIME                                                  | Uppdaterad                                                                    |

**Validering (affärsregler):**

*   Vid sparande/submit:
    *   `pay_rate_id` sätts genom lookup av giltig `pay_rates` för `(trainer_id, pay_group_id, date)` och **fryses**.
    *   Om `work_type='training_group'` → `work_ref_id` **måste** finnas (gruppens id).
    *   Om `work_type≠'training_group'` → `purpose` **måste** finnas (fritext: t.ex. tävling/utbildning).
*   Attestflöde: `draft → submitted → approved/rejected`.

**Index:**

*   `INDEX(trainer_id, date)`
*   `INDEX(pay_group_id)`
*   `INDEX(pay_rate_id)`
*   `INDEX(status)`
*   `INDEX(work_type, work_ref_id)`

> Tabellen täcker både träning och övriga uppdrag. Träningsgrupp refereras mjukt (endast id), allt annat beskrivs i `purpose`.

***

### `travel_reports`  *(resor – totalsummor per resa, inga dagsrader)*

| Fält            | Typ                                             | Beskrivning                   |
| --------------- | ----------------------------------------------- | ----------------------------- |
| id              | CHAR(36) PK                                     | UUID                          |
| trainer\_id     | CHAR(36) FK → `time_users.user_id`              | Vem resan avser               |
| start\_datetime | DATETIME                                        | Start                         |
| end\_datetime   | DATETIME                                        | Slut                          |
| country\_code   | CHAR(2)                                         | ISO‑land för traktamenteregel |
| purpose         | TEXT                                            | Resans avsikt                 |
| status          | ENUM('draft','submitted','approved','rejected') | Attestläge                    |

**Måltider & traktamente (summa per resa)**

| Fält                         | Typ           | Beskrivning                                                 |
| ---------------------------- | ------------- | ----------------------------------------------------------- |
| meals\_included\_breakfast   | TINYINT(1)    | 1/0                                                         |
| meals\_included\_lunch       | TINYINT(1)    | 1/0                                                         |
| meals\_included\_dinner      | TINYINT(1)    | 1/0                                                         |
| allowance\_amount            | DECIMAL(10,2) | **Beräknat traktamente (låst)**                             |
| allowance\_calculation\_meta | JSON NULL     | (Valfritt) Förklarande metadata (t.ex. hel/halvdag, avdrag) |

**Milersättning (summa per resa)**

| Fält            | Typ                | Beskrivning                       |
| --------------- | ------------------ | --------------------------------- |
| kilometers      | DECIMAL(8,1) NULL  | Körda km                          |
| vehicle\_type   | ENUM('car')        | Fordonstyp (kan utökas senare)    |
| mileage\_amount | DECIMAL(10,2) NULL | **Beräknad milersättning (låst)** |

**Övriga kostnader (summa per resa)**

| Fält                      | Typ                | Beskrivning                  |
| ------------------------- | ------------------ | ---------------------------- |
| other\_costs\_amount      | DECIMAL(10,2) NULL | Totalbelopp (utan kvittofil) |
| other\_costs\_description | TEXT NULL          | Kort beskrivning             |

**Tekniska fält**

| Fält        | Typ      |Beskrivning                  |
| ----------- | -------- |---------------------------- |
| exported_at     | DATETIME NULL | När posten har exporterats                                                    |
| export_batch_id | CHAR(36) NULL | Batch id för export                                                           |
| export_attempts | INT DEFAULT 0 | Antal försök till export                                                      |
| export_error    | TEXT NULL| Orsak till exporten misslyckades                                              |
| created\_at | DATETIME |
| updated\_at | DATETIME |

**Index:**

*   `INDEX(trainer_id, start_datetime)`
*   `INDEX(status)`

**Affärsregler (beräkning & frysning):**

*   **Traktamente:** beräknas vid sparande/submit utifrån `start/end` + `country_code` + `meals_*` + `travel_allowance_rules`. Summan sparas i `allowance_amount`.
    *   Enkel policy för årsskifte: använd regel för `YEAR(start_datetime)`; resor som korsar årsskifte kan vid behov delas upp av användaren.
*   **Milersättning:** `mileage_amount = kilometers × amount_per_km` enligt `mileage_rules` för `YEAR(start_datetime)` och `vehicle_type`. Summan sparas.
*   **Övriga kostnader:** en totalsumma + beskrivning. Kvittohantering sker utanför systemet.

***

### `travel_allowance_rules`  *(traktamente per land/år)*

| Fält              | Typ           | Beskrivning |
| ----------------- | ------------- | ----------- |
| id                | CHAR(36) PK   | UUID        |
| country\_code     | CHAR(2)       | ISO‑land    |
| year              | YEAR          | Gäller år   |
| full\_day         | DECIMAL(10,2) | Heldag      |
| half\_day         | DECIMAL(10,2) | Halvdag     |
| breakfast\_deduct | DECIMAL(10,2) | Avdrag      |
| lunch\_deduct     | DECIMAL(10,2) | Avdrag      |
| dinner\_deduct    | DECIMAL(10,2) | Avdrag      |
| created\_at       | DATETIME      | Skapad      |

**Index:** `UNIQUE(country_code, year)`

***

### `mileage_rules`  *(milersättning per år/fordon)*

| Fält            | Typ           | Beskrivning       |
| --------------- | ------------- | ----------------- |
| id              | CHAR(36) PK   | UUID              |
| year            | YEAR          | Gäller år         |
| vehicle\_type   | ENUM('car')   | Fordonstyp        |
| amount\_per\_km | DECIMAL(10,2) | Ersättning per km |
| created\_at     | DATETIME      | Skapad            |

**Index:** `UNIQUE(year, vehicle_type)`

***

### Statusflöden (översikt)

*   **Arbetspass:** `draft → submitted → approved/rejected`
*   **Resor:** `draft → submitted → approved/rejected`

Attest sker i Main UI. Godkända poster utgör löneunderlag.

***

### Relationsskiss (text)

*   `time_users (1)` —— `(N) pay_rates`
*   `pay_groups (1)` —— `(N) pay_rates`
*   `time_users (1)` —— `(N) time_entries`
*   `time_users (1)` —— `(N) travel_reports`
*   `travel_allowance_rules (1)` —— *(beräknar)* → `travel_reports.allowance_amount`
*   `mileage_rules (1)` —— *(beräknar)* → `travel_reports.mileage_amount`

> **Notera:** `work_ref_id` i `time_entries` är en **mjuk referens** till Group‑service (ingen DB‑FK över tjänstgräns).

***

### Designmotivering – sammanfattning

*   **Förenklad reserapportering:** inga dagsrader; totalsummor per resa med frysning.
*   **Förenklad tidsrapportering:** en tabell för både träning och övriga uppdrag; träningsgrupp via mjuk referens, övrigt via fritext.
*   **Korrekt ansvarsfördelning:** pay‑kategorier i TimeDB; träningsgrupper i Group‑service; tävlingar i Competition‑service.
*   **Säker historik:** `pay_rate_id`, `allowance_amount`, `mileage_amount` sparas och ändras inte retroaktivt.
*   **Enkel attest:** statusfält och snapshotade värden räcker för beslutsunderlag.

***

## 5.3. Results DB (Preliminär)

### Tabeller:

*   athletes
*   athlete\_name\_history
*   athlete\_license\_history
*   results (snapshot av namn/licens)

### Viktigt:

**Atleter ≠ användare**  
En atlet kopplas till en användare *endast om de loggar in* via fältet:

    athletes.user_id (nullable)

### Versionering:

*   Namn → athlete\_name\_history
*   Licens → athlete\_license\_history
*   Resultat sparar snapshot → historiskt korrekt

***

## 5.4. Competition DB (Preliminär)

### Tabeller:

*   competitions
*   competition\_events
*   entries

Kopplar entries → athletes.id (inte auth.users.id)

***

## 5.5 Group DB – Datamodell 

GroupDB tillhandahåller en central och enkel representation av **nuvarande träningsgrupper**.  
Den hanterar:

*   gruppernas namn och metadata
*   nuvarande gruppmedlemmar (aktiva)
*   nuvarande gruppledare/assistenter
*   stöder tidsrapporteringen (träningstid → work\_ref\_id)
*   stöder tävlingsanmälan (ledare kan anmäla aktiva ur sina grupper)

***

### Tabellöversikt

*   `groups` — gruppdefinitioner
*   `group_members` — nuvarande aktiva i gruppen
*   `group_leaders` — nuvarande ledare/assistenter i gruppen

***

### `groups`

Representerar en träningsgrupp.

| Fält        | Typ               | Beskrivning                            |
| ----------- | ----------------- | -------------------------------------- |
| id          | CHAR(36) PK       | UUID                                   |
| name        | VARCHAR(255)      | Gruppnamn, t.ex. “Friidrott Blå 2012”  |
| active      | TINYINT(1)        | 1 = aktiv grupp, 0 = inaktiv           |
| created\_at | DATETIME          | Skapad                                 |
| updated\_at | DATETIME          | Senast ändrad                          |

**Index:**

*   `INDEX(active)`
*   `INDEX(name)` *(snabb lookup från UI)*

***

### `group_members`

Aktiva som tillhör gruppen.  
Ingen historik – detta är den *aktuella* medlemslistan.

| Fält        | Typ                                 | Beskrivning |
| ----------- | ----------------------------------- | ----------- |
| group\_id   | CHAR(36) FK → `groups.id`           | Grupp       |
| athlete\_id | CHAR(36) FK → `results.athletes.id` | Aktiv       |

**Primärnyckel:**  
`PRIMARY KEY (group_id, athlete_id)`

**Index:**

*   `INDEX(athlete_id)` *(t.ex. visa alla grupper en aktiv tillhör)*

> **Anmärkning:**  
> `athlete_id` kommer från Results‑service, eftersom atleter är domänobjekt där.

***

### `group_leaders`

Ledare och assistenter som är kopplade till gruppen.  
Inte historiskt – visar *nuvarande* ansvariga.

| Fält      | Typ                           | Beskrivning              |
| --------- | ----------------------------- | ------------------------ |
| group\_id | CHAR(36) FK → `groups.id`     | Grupp                    |
| user\_id  | CHAR(36) FK → `auth.users.id` | Ledare/assistent         |
| role      | ENUM('leader','assistant')    | Typ av uppdrag i gruppen |

**Primärnyckel:**  
`PRIMARY KEY (group_id, user_id)`

**Index:**

*   `INDEX(user_id)`
*   `INDEX(group_id, role)`

> **Anmärkning:**  
> `user_id` kommer från Auth‑service → korrekt separation av identitet och verksamhet.

***

### Interaktioner med andra tjänster

#### Time‑service

*   När ledare rapporterar ett träningspass:  
    `time_entries.work_type = 'training_group'`  
    `time_entries.work_ref_id = groups.id`

*   Val före lagring: UI hämtar **grupper för aktuell ledare** via Group‑service (via group\_leaders).

#### Competition‑service

*   När ledare ska anmäla aktiva till tävlingar:  
    Group‑service returnerar **aktuell medlemslista** i gruppen.

*   Inga historikproblem eftersom anmälningar gäller “nuvarande aktiva”.

#### Results‑service

*   group\_members refererar `athletes.id`
*   inga dubbletter, enkel lookup

***

## Kort relationsskiss (text)

*   `groups (1)` —— `(N) group_members` (mot athletes)
*   `groups (1)` —— `(N) group_leaders` (mot users)
*   `group_members` hänvisar till `results.athletes`
*   `group_leaders` hänvisar till `auth.users`
*   `time_entries.work_ref_id` pekar mjukt på `groups.id` när `work_type='training_group'`

***

# 6. Arbetsflöden

## 6.1 Magic Link Login-flöde

1.  Användaren anger e‑post
2.  Auth-service skapar magic link
3.  Mail skickas
4.  Klick → validering
5.  JWT + refresh returneras
6.  Klienten dirigeras till respektive UI

***

## 6.2 Tidsrapportering 

1.  **Ledaren rapporterar arbetspass**
    *   anger datum, minuter, ersättningskategori (`pay_group`)
    *   anger arbetets typ (training\_group / competition / education / other)
    *   vid träning väljer ledaren grupp (`work_ref_id`)
    *   vid övriga uppdrag skriver ledaren en fritext (“purpose”)

2.  **Systemet hämtar gällande timlön**
    *   Time‑service söker korrekt `pay_rates` för `(trainer, pay_group, date)`
    *   `pay_rate_id` sätts i `time_entries` och **fryser historiken**

3.  **Ledaren skickar in (status: submitted)**
    *   ytterligare validering kan göras automatiskt
    *   t.ex. att gruppid finns, att purpose är ifyllt vid rätt work\_type, etc.

4.  **Admin attesterar i Main UI**
    *   status ändras till `approved` eller `rejected`

5.  **Godkända poster låses**
    *   `time_entries` kan inte längre ändras
    *   utgör löneunderlag för utbetalning

***

## 6.3 Reseräkning

1.  **Ledaren skapar en reserapport (`travel_report`)**
    *   anger startdatum + tid
    *   anger slutdatum + tid
    *   anger land
    *   beskriver resans avsikt (purpose)

2.  **Ledaren fyller i måltider**
    *   frukost / lunch / middag (ja/nej per måltid)
    *   dessa används i beräkningsregeln

3.  **Ledaren fyller i milersättning**
    *   antal kilometer
    *   fordonstyp (oftast alltid `car`)
    *   systemet kommer använda årets regel

4.  **Ledaren anger övriga kostnader (valfritt)**
    *   totalbelopp (other\_costs\_amount)
    *   kort beskrivning (other\_costs\_description)

5.  **Systemet beräknar och fryser ersättningarna**
    *   **traktamente** från `travel_allowance_rules` (per land & år)
        *   heldag/halvdag beräknas automatiskt från datumen
        *   måltidsavdrag appliceras
        *   sparas som `allowance_amount`
    *   **milersättning** från `mileage_rules` (per år & fordonstyp)
        *   `mileage_amount = kilometers × amount_per_km`
        *   sparas som `mileage_amount`

6.  **Inga kvitton laddas upp**
    *   hanteras utanför systemet (enligt verksamhetsprocess)

7.  **Ledaren skickar in (status: submitted)**
    *   beräkningarna har redan frysts
    *   kansliet kan ändå justera manuellt vid behov

8.  **Admin attesterar (status: approved/rejected)**
    *   godkända poster är låsta underlag för arvode


***

## 6.4 Resultathantering

1.  Import eller registrering
2.  Historiska fält snapshotas
3.  Visas i PWA Aktiva och Main UI

***

# 7. CI/CD – Pipelines & Deployment

## 7.1 GitHub Actions pipelines

Per tjänst:

    composer install
    phpunit
    docker build
    docker push

## 7.2 Deploy

På produktionsservern:

    docker compose pull
    docker compose up -d
    phinx migrate

## 7.3 Databasmigrering

*   Phinx per tjänst
*   Seeds för auth (roller/permissions) idempotenta
*   Aldrig cross‑service migrations

***

# 8. Driftsmiljö

*   Docker Compose i produktion
*   MariaDB som värdtjänst (ej container)
*   Backup via `mysqldump`
*   Health checks via gateway
*   Logging → Monolog → filer/syslog
*   Rotation med logrotate


***

# 9. Komponenter och kodstruktur (C4 Model: Level 3 — Slim Action‑baserad)

Detta kapitel beskriver hur varje mikrotjänst är strukturerad enligt **Slims action‑baserade arkitektur**:

*   *En route = en Action‑klass (invokable handler)*
*   Actions är tunna och hanterar endast request/response
*   Services hanterar affärslogik
*   Repositories hanterar datalagring
*   Domain‑objekt beskriver affärsmodeller

Strukturen gäller konsekvent i alla tjänster.

***

## 9.1 Auth‑service

Auth‑service ansvarar för identitet, autentisering (passwordless), tokenhantering och roll/behörighetsmodell.  
Varje endpoint i Auth‑service implementeras som en **invokable Action‑klass** enligt Slims action‑baserade arkitektur.  
Ingen domändata lagras här — endast identitets‑ och säkerhetsdata.

***

### Actions (invokable handlers)

#### Magic‑link (lösenordslös inloggning)

*   `RequestMagicLinkAction`  
    Tar emot e‑postadress, genererar magic‑link och skickar (ex via mailservice).
*   `ConsumeMagicLinkAction`  
    Validerar token, konsumerar länken, utfärdar access- och refresh‑token.

#### Refresh & tokenförnyelse

*   `GenerateTokenAction`  
    Byter refresh‑token mot ny access‑token (och ev. ny refresh‑token).

#### OTP (engångskoder)

*   `RequestOtpAction`  
    Skapar engångskod och skickar (via e‑post eller annan kanal).
*   `ConsumeOtpAction`  
    Validerar OTP‑kod och utfärdar tokens.

#### TOTP (tvåfaktorsinloggning, för t.ex. admin/attest)

*   `EnableTotpAction` *(valfritt i v1)*  
    Genererar hemlighet & QR‑kod för aktivering.
*   `VerifyTotpAction`  
    Verifierar TOTP‑kod vid känsliga operationer.

#### Roller och rättigheter (admin

*   `CreateRoleAction`
*   `AssignRoleAction`
*   `CreatePermissionAction`
*   `AssignRolePermissionAction`  
    *(Exponeras endast för systemadministratörer.)*

***

### Services (affärslogik)

#### MagicLinkService

*   Generering av unika tokens
*   Hashning och lagring
*   Validering av token (tid, client\_type, konsumerad/okonsumerad)
*   Konsumtion (sätta `consumed_at`)
*   Utlösning av mailutskick via mail‑adapter

#### OtpService

*   Skapande och validering av OTP (engångskoder)
*   Hashning av kod
*   Tidbegränsning
*   “One‑use only”‑principen säkerställs

#### TotpService

*   Skapande och validering av TOTP
*   Hantering av TOTP‑hemligheter i `totp_secrets`
*   Device‑independent 2FA
*   Skydd för adminroller och attestanter

#### TokenService

*   Generering av JWT access‑ och refresh‑tokens
*   Validering av token‑claims
*   Hantering av `client_type`‑claim
*   Produktion av signering (HS256 eller ES256)
*   Rotation av refresh‑tokens (om aktiverat)

#### UserService

*   Skapande av användare i `users` (om det inte finns en sedan tidigare)
*   Uppdatering av email/namn
*   Hämtning av user‑objekt
*   Hjälpfunktioner för lookup av roller/permissions

#### RoleService / PermissionService

*(i vissa team läggs dessa i samma klass — du avgör själv nivå av separation)*

*   Skapande av roller och permissions
*   Kopplingar via user\_roles och role\_permissions
*   Framtida export för Gateway‑policyer (t.ex. RBAC‑cache)

***

### Repositories (databasåtkomst via Doctrine DBAL)

#### Kärn‑repositories

*   `UserRepository`  
    Hämtar, skapar och uppdaterar användare (ingen domändata).

*   `MagicLinkRepository`  
    Lagrar hashade magic‑link‑tokens + expiry + consumed\_at.

*   `OtpRepository`  
    Lagrar engångskoder (hash + expiry).

*   `TotpSecretRepository`  
    Lagrar TOTP‑hemligheter per user.

#### RBAC‑repositories

*   `RoleRepository`
*   `PermissionRepository`
*   `UserRoleRepository`
*   `RolePermissionRepository`

> **Notera:**  
> Auth‑service håller RBAC‑modellen i sin helhet. Andra tjänster ska *aldrig* göra RBAC‑joins – de får JWT‑claims från Gateway.

***

### Domain‑modeller

Domänmodellerna i Auth‑service är lätta och representerar endast säkerhets‑ och identitetsdata.

*   `User`  
    (id, email, first\_name, last\_name, created\_at, updated\_at)

*   `Role`

*   `Permission`

*   `UserRole`

*   `RolePermission`

*   `MagicLink`  
    (id, user\_id, token\_hash, client\_type, expires\_at, consumed\_at)

*   `OneTimeCode`  
    (id, user\_id, code\_hash, expires\_at, consumed\_at)

*   `TotpSecret`  
    (user\_id, secret, created\_at, last\_used\_at)


***

### Sammanfattning

Auth‑service består av:

*   **Actions** för:
    *   magic‑link
    *   OTP
    *   TOTP
    *   tokenförnyelse
    *   RBAC‑administration

*   **Services** för:
    *   identitet
    *   passwordless authentication
    *   tokenhantering
    *   RBAC‑logik
    *   TOTP‑skydd

*   **Repositories** för:
    *   users
    *   magic links
    *   OTP
    *   TOTP
    *   roller/permissions + kopplingar

*   **Domänmodeller** som strikt speglar AuthDB‑tabellerna.

Detta är en **ren, skalbar och mikrotjänstanpassad Auth‑tjänst**, perfekt för lösenordslöst login och rollbaserad åtkomst till dina övriga tjänster.

***

## 9.2 Time‑service

Time‑service ansvarar för all tidsrapportering, reserapportering samt datumstyrda ersättningsregler.  
Arkitekturen följer Slims **Action‑baserade modell**: varje endpoint representeras av en invokable Action‑klass.

Domänlogiken bor i Services och datalagringen i Repositories.

***

### Actions (invokable handlers)

Uppdaterad lista baserat på aktuell funktionalitet:

#### Tidsrapportering

*   `CreateTimeEntryAction` – skapa nytt arbetspass
*   `SubmitTimeEntryAction` – skicka tidrapport (draft → submitted)
*   `ApproveTimeEntryAction` – attestera arbetspass
*   `RejectTimeEntryAction` – avslå arbetspass

#### Reseräkningar

*   `CreateTravelReportAction` – skapa resa
*   `SubmitTravelReportAction` – ändra status till submitted
*   `ApproveTravelReportAction` – attestera resa
*   `RejectTravelReportAction` – avslå resa

*(Resor skapas nu helt utan travel\_days och mileage\_entries — allt görs på resenivå.)*

#### Regelverk (admin)

*   `CreatePayRateAction` – registrera ny period (versionerad timlön)
*   `CreateAllowanceRuleAction` – lägga till traktamenteregel (land/år)
*   `CreateMileageRuleAction` – lägga till milregel (år/typ)

#### Export

*   `ExportTimeEntriesAction` – exportera attesterad arbetstid
*   `ExportTravelReportsAction` – exportera attesterade resor

***

### Services (domänlogik)

#### TimeEntryService

*   Hanterar skapande, uppdatering och attestering av arbetspass
*   Innehåller logik för work\_type, work\_ref\_id och purpose
*   Validerar obligatoriska fält beroende på typ
*   Sätter status och exportfält

#### PayRateService

*   Svarar för rate‑lookup:
    *   hämtar korrekt `pay_rate` för (ledare, pay\_group, datum)
    *   sätter `pay_rate_id` i `time_entries` vid sparande
*   Säkerställer historik genom att rate alltid fryses

#### TravelReportService

*   Skapar och uppdaterar resor (inkl. syfte, måltider, km, övriga kostnader)
*   Hanterar statusflödet (`draft → submitted → approved/rejected`)
*   Sätter exportstatus

#### AllowanceCalculationService

Beräknar *total* traktamentsersättning baserat på:

*   start/slut (DATETIME)
*   land
*   måltidsavdrag
*   regler för `(country_code, year)`  
    **Resultat:** `allowance_amount` + valfri `allowance_calculation_meta`

#### MileageCalculationService

Beräknar total milersättning:

*   tar kilometer
*   slår upp aktuell regel för `(year, vehicle_type='car')`
*   multiplicerar
*   returnerar **låst** `mileage_amount`

#### ExportService

*   Filtrerar `approved` + `exported_at IS NULL`
*   Sköter exportlogik mot lönehantering
*   Skriver: `exported_at`, `export_batch_id`, `export_error`, `export_attempts`

***

### Repositories (databasåtkomst, Doctrine DBAL)

#### Huvud‑repositories

*   `TimeEntryRepository` – CRUD för arbetspass
*   `TravelReportRepository` – CRUD för resor
*   `PayRateRepository` – versionerad timlön
*   `PayGroupRepository` – ersättningskategorier
*   `AllowanceRuleRepository` – traktamenteregler per land/år
*   `MileageRuleRepository` – milersättningsregler per år/fordon
*   `TimeUserRepository` – verksamhetsdata för ledare

### Domain‑modeller

Domänobjekten representerar kärnstrukturerna i Time‑service.

*   `TimeEntry`
    *   innehåller rate‑ID, work\_type, work\_ref\_id, purpose, status, exportstatus
*   `TimeUser`
*   `PayRate`
*   `PayGroup`
*   `TravelReport`
    *   innehåller traktamente, milersättning, övriga kostnader (totalbelopp)
*   `TravelAllowanceRule`
*   `MileageRule`

***

## 9.3 Results‑service

**Actions**

*   `CreateAthleteAction`
*   `UpdateAthleteAction`
*   `GetAthleteAction`
*   `ListAthletesAction`
*   `RecordResultAction`
*   `ImportResultsAction`
*   `GetResultsForAthleteAction`
*   `ListTopResultsAction`

**Services**

*   `AthleteService` — hantering av domänobjektet “atlet”
*   `HistoryService` — hantering av namn-/licenshistorik
*   `SnapshotService` — snapshotting av namn/licens vid registrering
*   `ResultService` — resultatlogik, PB/ÅB‑beräkning
*   `ImportService` — fil/importflöden

**Repositories**

*   `AthleteRepository`
*   `AthleteNameHistoryRepository`
*   `AthleteLicenseHistoryRepository`
*   `ResultRepository`

**Domain‑modeller**

*   `Athlete`
*   `AthleteNameHistory`
*   `AthleteLicenseHistory`
*   `Result`

**Princip för results-service**

*   `Athlete.id` är domän-ID
*   `athlete.user_id` är *nullable* och sätts först vid login
*   snapshot-fält i `Result` borgar för historisk korrekthet

***

## 9.4 Competition‑service

**Actions**

*   `CreateCompetitionAction`
*   `UpdateCompetitionAction`
*   `ListCompetitionsAction`
*   `CreateEventAction`
*   `ListEventsAction`
*   `RegisterEntryAction`
*   `ListEntriesAction`

**Services**

*   `CompetitionService`
*   `EventService`
*   `EntryService`

**Repositories**

*   `CompetitionRepository`
*   `EventRepository`
*   `EntryRepository`

**Domain‑modeller**

*   `Competition`
*   `CompetitionEvent`
*   `Entry`

***

## 9.5 Group‑service

Group‑service ansvarar för systemets **träningsgrupper**, inklusive nuvarande aktiva och nuvarande tränare/assistenter.  
Tjänsten tillhandahåller en enkel, stabil och mikrotjänstanpassad API‑yta som:

*   UI använder för att ledare ska välja grupp vid tidsrapportering
*   Time‑service refererar via `work_ref_id` när `work_type='training_group'`
*   Competition‑service använder när ledare ska anmäla aktiva till tävlingar

***

### Actions (invokable handlers)

#### Grupper

*   `ListGroupsAction`  
    Returnerar alla aktiva grupper (för UI‑listor, t.ex. vid tidsrapportering).
*   `GetGroupAction`  
    Hämtar detaljer om en specifik grupp.
*   `CreateGroupAction`  
    Skapar en ny grupp.
*   `UpdateGroupAction`  
    Uppdaterar namn, beskrivning, sport eller active/inactive‑status.

#### Gruppmedlemmar (aktiva)

*   `ListGroupMembersAction`  
    Lista nuvarande medlemmar (atleter) i en grupp.
*   `AddGroupMemberAction`  
    Lägg till en aktiv i gruppen.
*   `RemoveGroupMemberAction`  
    Ta bort en aktiv från gruppen.

#### Gruppledare (leader/assistenter)

*   `ListGroupLeadersAction`  
    Lista ledare och assistenter för en grupp.
*   `AddGroupLeaderAction`  
    Koppla en ledare/assistent till gruppen.
*   `RemoveGroupLeaderAction`  
    Ta bort en ledare/assistent från gruppen.

***

### Services (affärslogik)

#### GroupService

*   Skapar, uppdaterar och hämtar grupper
*   Hanterar active/inactive‑status
*   Lägger till eller tar bort gruppmedlemmar
*   Lägger till eller tar bort gruppledare
*   Säkerställer att samma user/athlete inte läggs till två gånger
*   Säkerställer att endast aktiva grupper returneras vid listning för UI

#### GroupMembershipService

*   Inkapslar logik kring vilka aktiva som är medlemmar
*   Kan användas av Competition‑service för att validera anmälningar
*   Kan användas av Time‑service (valfritt) för substitutionskontroll i framtiden

#### GroupLeaderService

*   Hanterar ledare/assistenter i grupper
*   Används av UI för att visa vilken ledare som har vilka grupper
*   Används av Time‑service: UI kan lista *endast de grupper en viss ledare ansvarar för*

***

### Repositories (databasåtkomst via Doctrine DBAL)

*   `GroupRepository`  
    CRUD för `groups`

*   `GroupMemberRepository`  
    Hanterar relationen mellan grupper och atleter (`group_members`)

*   `GroupLeaderRepository`  
    Hanterar relationen mellan grupper och användare (`group_leaders`)

Alla repositories är tunn datalagringslogik utan affärsregler — all domänlogik ligger i *services*.

***

### Domain‑modeller

Domänobjekten i Group‑service motsvarar datamodellen i GroupDB:

*   `Group`  
    (id, name, description, sport, active, created\_at, updated\_at)

*   `GroupMember`  
    (group\_id, athlete\_id)

*   `GroupLeader`  
    (group\_id, user\_id, role)

Dessa modeller är avsedda för internt bruk; API‑svaren kan presentera koncisa DTOs.

***

### Integrationer med andra tjänster

#### Time‑service

*   Vid rapportering av träningspass anger UI:
    *   `work_type = 'training_group'`
    *   `work_ref_id = group.id`
*   Time‑service gör inga joiner — endast mjuk referens.

#### Competition‑service

*   Vid tävlingsanmälan hämtas:
    *   gruppens aktuella aktiva (`group_members`)
    *   gruppens ledare (via `group_leaders`)
*   Ger korrekt UI‑listning även om tävlingen registreras långt i förväg.

#### Results‑service

*   Group‑service använder `athlete_id` från Results‑service.
*   Ingen direktkoppling åt andra hållet.

#### Auth‑service

*   Gruppledare refererar `user_id` från AuthDB.
*   Group‑service hanterar ingen identitetsinformation.

***

### Designmotivering – sammanfattning

*   **Minimal men tillräcklig modell:** ingen historik, endast aktuella relationer.
*   **Mikrotjänstseparation:** Idrottsstruktur hanteras isolerat från tidrapporter och resor.
*   **Flexibelt för UI:** grupper används vid tidrapport, tävlingsanmälan, och ledaröversikt.
*   **Enkel att expandera i framtiden:** historik, scheman och nivåstruktur kan läggas till vid behov.
*   **Mjuk koppling:** andra tjänster använder endast `group_id`, aldrig direkta databaskopplingar.

***


# 10. Säkerhet – detaljer

*   CORS whitelist per klienttyp
*   Begränsade JWT lifetimes
*   Refresh tokens bundna till user + client
*   Blocklistning vid logout
*   Alla tokens signerade med HS256 eller ES256
*   Rate limiting i gateway
*   Intern kommunikation signerad med X‑Service‑Key


***

# 11. API‑kontrakt och OpenAPI‑specifikation

Detta kapitel beskriver hur systemets API:er dokumenteras, valideras och används genom en **OpenAPI‑specifikation**. OpenAPI fungerar som ett *kontrakt* mellan backend och frontend och säkerställer att alla mikrotjänster och klienter har en gemensam, maskinläsbar beskrivning av API‑formatet.

OpenAPI är inte ett verktyg utan en **standard (OAS)** som definierar ett API på ett strukturerat sätt i en `.yaml`‑ eller `.json`‑fil. Denna fil kan sedan ligga till grund för dokumentation, klientgenerering, kontraktstester och kvalitetssäkring i CI/CD.

***

## 11.1 Syftet med ett API‑kontrakt

API‑kontrakten har tre huvudsyften:

### 1) Tydlig kommunikation

Alla utvecklare — backend, gateway, frontend, externa integratörer — arbetar mot **samma definition** av API:et. Detta eliminerar tvetydigheter i datamodeller, felkoder och endpoints.

### 2) Programmatisk garanti

Maskinläsbara kontrakt betyder att:

*   frontend kan generera typade klienter automatiskt
*   CI/CD kan testa att API:er inte brutits av misstag
*   dokumentation hålls **automatiskt uppdaterad**

### 3) Långsiktig stabilitet

När API:er versioneras eller utvecklas vidare säkerställs kompatibilitet och bakåtkompatibilitet via kontraktstester.

***

## 11.2 OpenAPI‑specifikationens innehåll

Varje mikrotjänst har en egen `openapi.yaml`, men Gateway‑API kan exponera en **aggregerad** version om så önskas (valfritt).

En OpenAPI‑fil innehåller följande delar:

### 11.2.1 Paths – API‑endpoints

Alla HTTP‑rutter definieras med:

*   metod (`GET`, `POST`, `PUT`, `DELETE` …)
*   parametrar (path/query/header/body)
*   request‑schema
*   response‑schema
*   felkoder

Exempel:

```yaml
paths:
  /results/{athleteId}:
    get:
      summary: Hämta resultat för en atlet
      parameters:
        - in: path
          name: athleteId
          required: true
          schema:
            type: string
      responses:
        "200":
          description: Lyckad hämtning
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/AthleteResults"
        "404":
          description: Atlet saknas
```

***

### 11.2.2 Components – datamodeller

Här definieras de gemensamma objekttyper som används i request/response:

```yaml
components:
  schemas:
    Athlete:
      type: object
      properties:
        id:
          type: string
        currentFirstName:
          type: string
        currentLastName:
          type: string
        gender:
          type: string
        birthdate:
          type: string
          format: date
```

Modellerna motsvarar *domänobjekten* i backend men är **kontraktsmässiga representationer**, inte interna PHP‑klasser.

***

### 11.2.3 Security – JWT och klientpolicy

OpenAPI beskriver hur klienter autentiseras och vilka headers de måste skicka:

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - bearerAuth: []
```

Det är även här `client_type` i JWT dokumenteras.

***

### 11.2.4 Felhantering

Standardiserade fel gör API:er mer förutsägbara:

```yaml
"400":
  description: Ogiltig begäran
  content:
    application/json:
      schema:
        $ref: "#/components/schemas/ErrorResponse"
```

Detta används för att beskriva:

*   valideringsfel
*   behörighetsfel
*   domänfel

***

## 11.3 Genererade klienter (TypeScript m.fl.)

OpenAPI används för att generera **type‑safe API‑klienter** till dina frontends.

Dina tre frontends (PWA Tränare, PWA Aktiva, Main UI) kan använda automatiskt genererade klienter som innehåller:

*   datamodeller som `Athlete`, `Result`, `TimeEntry`, `TravelReport`
*   functions med korrekt typning:  
    `getAthleteResults(id: string): Promise<AthleteResults>`
*   feltyper och validering

Vanliga generatorer:

*   `openapi-typescript`
*   `openapi-generator-cli`
*   `swagger-typescript-api`

Detta eliminerar duplikation av modeller i frontend och minskar fel.

***

## 11.4 Mock‑servrar

OpenAPI möjliggör mockning av API:er innan backend är färdig, så frontend kan utvecklas parallellt.

Exempel:

```bash
npx openapi-mock-server openapi.yaml
```

Frontend kan då:

*   utvecklas utan att backend är klar
*   testa UI‑flöden med realistiska svar
*   undvika hårdkodade stubbar

Detta accelererar utvecklingsprocessen avsevärt.

***

## 11.5 CI/CD‑validering (kontraktsstyrd utveckling)

OpenAPI används i CI/CD för att upptäcka brutna ändringar innan de påverkar klienterna.

Steg inkluderar:

1.  **OpenAPI‑linter**
    *   kontrollerar att filen följer standard
2.  **Schema‑validering**
    *   säkerställer att alla endpoints har giltiga schema
3.  **breaking-change detection**  
    – varnar om ett API bryter kompatibilitet
4.  **generera TypeScript‑klient i CI**  
    – bekräftar att modellen är korrekt

Verktyg:

*   openapi-cli
*   swagger-cli
*   speccy
*   openapi-diff (jämför äldre versioner)

Detta integreras i samma pipeline som PHPUnit, PHPStan och övriga verktyg.

***

## 11.6 Versionering av API‑kontrakt

API:er versioneras normalt per tjänst:

    /openapi/v1/auth.yaml
    /openapi/v1/time.yaml
    /openapi/v1/results.yaml
    /openapi/v1/competition.yaml

Rekommenderad strategi:

*   Brytande ändringar → **ny API‑version** (v2)
*   Mindre ändringar → behåll v1
*   Gateway kan mappa `/api/v1/...` mot rätt tjänst
*   Klienter väljer aktiv version vid uppgradering

Detta ger frihet att utveckla API:et utan att störa befintliga klienter.

***

## 11.7 OpenAPI i mikrotjänstkontexten

Eftersom varje tjänst är frikopplad:

*   varje tjänst har **sin egen** OpenAPI‑fil
*   gatewayn kan sammanfoga dem virtuellt (om önskas)
*   frontend får en **genererad klient per tjänst**

Det leder till:

*   hög stabilitet
*   lätta förändringar
*   tydlig ansvarsfördelning
*   möjlig framtida polyglot‑backend (PHP, Go, Python etc.)


***

## 11.8 DTO‑modeller (Request/Response)

DTO:er (Data Transfer Objects) definierar **exakta payloads** som frontend och backend utbyter via API:erna i Auth‑, Time‑ och Group‑service.

Dessa DTO‑er används:

*   för att generera OpenAPI‑specifikation
*   för att autogenerera TypeScript‑klienter
*   för strikt kontraktsstyrning
*   som stabila modeller i alla actions (input/output)
*   som gränssnitt mot UI

DTO:erna i detta kapitel är **stabila API‑kontrakt** — inte databasmodeller.

***

### 11.8.1 Auth‑service DTOs

Auth‑service är lösenordslös (magic‑link/OTP/TOTP).  
Nedan följer alla centrala DTO‑er.

***

#### Request DTOs

##### RequestMagicLinkRequest

Användaren begär en magic‑link för inloggning.

```json
{
  "email": "user@example.com",
  "client_type": "pwa_trainer"
}
```

`client_type`: `"pwa_trainer" | "pwa_results" | "main_ui"`

***

##### ConsumeMagicLinkRequest

```json
{
  "token": "string"
}
```

***

##### RequestOtpRequest

```json
{
  "email": "user@example.com"
}
```

***

##### ConsumeOtpRequest

```json
{
  "email": "user@example.com",
  "code": "123456"
}
```

***

##### VerifyTotpRequest

```json
{
  "user_id": "uuid",
  "code": "123456"
}
```

***

##### RefreshTokenRequest

```json
{
}
```
`refresh_token` som cookie

***

#### Response DTOs

##### AuthTokenResponse

```json
{
  "access_token": "string",
  "token_type": "Bearer"
}
```
`refresh_token` som http-only cookie

***

##### UserProfileResponse

(returneras t.ex. efter succesful login)

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "first_name": "Anna",
  "last_name": "Karlsson",
  "roles": ["trainer"],
  "permissions": ["time.submit", "time.approve"]
}
```

***

### 11.8.2 Time‑service DTOs

Time‑service hanterar:

*   tidsrapportering
*   reseräkningar
*   rate‑beräkning
*   statusflöden
*   export

Nedan listas DTO‑er för arbetspass och resor.

***

#### A. Time Entry DTOs

##### CreateTimeEntryRequest

```json
{
  "date": "2026-03-10",
  "minutes": 90,
  "pay_group_id": "uuid",
  "work_type": "training_group",
  "work_ref_id": "uuid",
  "purpose": null,
  "description": "Hoppteknik"
}
```

`work_type`: `"training_group" | "competition" | "education" | "other"`

**Regler:**

*   Om `work_type="training_group"` → `work_ref_id` krävs
*   Annars → `purpose` krävs

***

##### TimeEntryResponse

(efter create/submit/approve)

```json
{
  "id": "uuid",
  "trainer_id": "uuid",
  "date": "2026-03-10",
  "minutes": 90,
  "pay_group_id": "uuid",
  "pay_rate_id": "uuid",
  "status": "submitted",
  "work_type": "training_group",
  "work_ref_id": "uuid",
  "purpose": null,
  "description": "Hoppteknik",
  "exported_at": null,
  "created_at": "2026-03-10T17:30:00Z",
  "updated_at": "2026-03-10T17:30:00Z"
}
```

***

##### Submit/Approve/RejectTimeEntryRequest

```json
{
  "id": "uuid",
  "new_status": "submitted"
}
```

***

#### B. Travel Report DTOs

##### CreateTravelReportRequest

```json
{
  "start_datetime": "2026-03-14T07:30:00",
  "end_datetime": "2026-03-14T22:15:00",
  "country_code": "FI",
  "purpose": "DM i Ekenäs",
  "meals_included": {
    "breakfast": false,
    "lunch": true,
    "dinner": false
  },
  "kilometers": 137.5,
  "vehicle_type": "car",
  "other_costs_amount": 12.50,
  "other_costs_description": "Parkeringsautomat"
}
```

***

##### TravelReportResponse

```json
{
  "id": "uuid",
  "trainer_id": "uuid",
  "start_datetime": "2026-03-14T07:30:00",
  "end_datetime": "2026-03-14T22:15:00",
  "country_code": "FI",
  "purpose": "DM i Ekenäs",
  "status": "submitted",

  "meals_included": {
    "breakfast": false,
    "lunch": true,
    "dinner": false
  },
  "allowance_amount": 34.00,
  "allowance_calculation_meta": {
    "full_days": 0,
    "half_day_departure": true,
    "half_day_return": false,
    "meal_deductions": {
      "lunch": true
    }
  },

  "kilometers": 137.5,
  "vehicle_type": "car",
  "mileage_amount": 69.00,

  "other_costs_amount": 12.50,
  "other_costs_description": "Parkeringsautomat",

  "exported_at": null,
  "created_at": "2026-03-14T22:30:00Z",
  "updated_at": "2026-03-14T22:30:00Z"
}
```

***

##### Submit/Approve/RejectTravelReportRequest

```json
{
  "id": "uuid",
  "new_status": "submitted"
}
```

***

### 11.8.3 Group‑service DTOs

Group‑service ansvarar för träningsgrupper, med **inga historiska kopplingar**.

***

#### A. Group DTOs

##### CreateGroupRequest

```json
{
  "name": "Friidrott Blå 2012",
  "description": "Hopp och sprint",
  "sport": "Athletics",
  "active": true
}
```

***

##### GroupResponse

```json
{
  "id": "uuid",
  "name": "Friidrott Blå 2012",
  "description": "Hopp och sprint",
  "sport": "Athletics",
  "active": true,
  "created_at": "2026-02-01T08:00:00Z",
  "updated_at": "2026-02-01T08:00:00Z"
}
```

***

#### B. Group Member DTOs

##### AddGroupMemberRequest

```json
{
  "group_id": "uuid",
  "athlete_id": "uuid"
}
```

***

##### GroupMemberResponse

```json
{
  "group_id": "uuid",
  "athlete_id": "uuid"
}
```

***

##### ListGroupMembersResponse

```json
{
  "group_id": "uuid",
  "members": [
    {
      "athlete_id": "uuid",
      "first_name": "Maja",
      "last_name": "Svensson"
    },
    {
      "athlete_id": "uuid",
      "first_name": "Ella",
      "last_name": "Virtanen"
    }
  ]
}
```

(*namnfälten hämtas från Results‑service, via snapshot in UI eller via sammansatt API‑förfrågan*)

***

#### C. Group Leader DTOs

##### AddGroupLeaderRequest

```json
{
  "group_id": "uuid",
  "user_id": "uuid",
  "role": "assistant"
}
```

***

##### GroupLeaderResponse

```json
{
  "group_id": "uuid",
  "user_id": "uuid",
  "role": "leader"
}
```

***

##### ListGroupLeadersResponse

```json
{
  "group_id": "uuid",
  "leaders": [
    {
      "user_id": "uuid",
      "first_name": "Anna",
      "last_name": "Karlsson",
      "role": "leader"
    },
    {
      "user_id": "uuid",
      "first_name": "Erik",
      "last_name": "Nyman",
      "role": "assistant"
    }
  ]
}
```

(*namnfälten hämtas från Auth‑service eller via Gateway*)

***

### 11.8.4 Designnoter för DTO‑modellerna

*   Alla DTO:er är **API‑kontrakt** — inte databastabeller.
*   DTO:erna mappar 1:1 mot OpenAPI‑specens schema‑sektion.
*   DTO:erna ska vara stabila och versionsstyrda per tjänst (se Kapitel 14).
*   DTO:erna är designade för att kunna generera:
    *   TypeScript‑klienter
    *   JSON‑schema
    *   API‑mockning för frontend
    *   automatiska valideringar i actions

DTO‑erna sammanfattar exakt vad UI behöver läsa och skicka — varken mer eller mindre.

***

### 11.8.5 Sammanfattning

Detta kapitel definierar samtliga centrala DTO‑er för:

*   **Auth‑service** (passwordless login och tokenhantering)
*   **Time‑service** (tidsrapportering och reseräkningar)
*   **Group‑service** (träningsgrupper, medlemmar och ledare)

Tillsammans bildar de kärnan i systemets **API‑kontrakt** och utgör grunden för OpenAPI‑specifikationer och TypeScript‑klienter.

***

## 11.9 Versionering och skydd av API‑kontrakt (DTO + OpenAPI)

API‑kontraktet, inklusive samtliga DTO‑er som definierar request‑ och response‑modeller för Auth‑, Time‑ och Group‑service, utgör en **stabil integrationsyta** mellan backend och frontend (PWA Tränare, PWA Aktiva, Main UI) samt mellan mikrotjänster.  
Det är därför avgörande att API‑kontraktet **inte ändras oavsiktligt**, och att alla ändringar är **spårbara**, **versionshanterade**, och **validerade**.

Detta kapitel beskriver hur systemet säkerställer att:

*   API‑kontrakten är **konsekventa**
*   DTO‑erna är **versionsstyrda**
*   Breaking changes **aldrig släpps av misstag**
*   CI/CD **stoppar felaktiga ändringar**
*   Backend och frontend **aldrig divergerar**

Metoderna nedan gäller för **alla tjänster** som har API‑ytor.

***

### 11.9.1 OpenAPI som “Single Source of Truth”

Varje tjänst har en egen `openapi.yaml` som innehåller:

*   alla endpoints
*   alla request‑DTO:er
*   alla response‑DTO:er
*   eventuella felmodeller
*   schema‑definitioner för alla dataobjekt

OpenAPI‑specifikationen är den **enda** källan som får beskriva API‑kontraktet.  
All annan representation (TypeScript‑klienter, backend‑validators, dokumentation) genereras från denna källa.

Fördelar:

*   ingen dubbeldefinition
*   ingen manuell typning i frontend
*   omöjligt för backend och frontend att “glida isär”
*   hög spårbarhet i Git

***

### 11.9.2 Autogenererade TypeScript‑klienter

Frontendprojekt (UI + PWAs) genererar klienter automatiskt från OpenAPI med verktyg som:

*   `openapi-typescript`
*   `openapi-generator-cli`
*   `swagger-typescript-api`

Resultat:

*   starkt typade klienter
*   inga manuella modeller
*   fullständig matchning mellan frontend och backend
*   noll risk för “felparameter” eller fel modell

Klienterna ska **aldrig skrivas manuellt** — de ska *alltid* genereras.

***

### 11.9.3 CI: OpenAPI‑linter (syntax & best practice)

CI/CD validerar varje ändring i OpenAPI‑filerna med:

*   `@redocly/openapi-cli lint`
*   `swagger-cli validate`

Det stoppar:

*   ogiltig YAML
*   saknade schema‑referenser
*   fel i response‑definitioner
*   fel i statuskoder
*   bruten struktur

Med andra ord: API‑specen **kan inte** brytas av misstag.

***

### 11.9.4 CI: Detect Breaking Changes (openapi-diff)

CI använder ett verktyg som `openapi-diff` för att jämföra PR‑ens OpenAPI‑spec mot:

*   senaste commit på main
*   tjänstens senaste release‑specifikation

`openapi-diff` klassificerar ändringar som:

*   **Breaking** (MAJOR)
*   **Non‑breaking additions** (MINOR)
*   **Fixes/metadata** (PATCH)

Om en breaking change detekteras:

→ **CI stoppar PR**  
→ utvecklaren måste **bumpa VERSION‑filen** från  
`X.Y.Z` → `(X+1).0.0`  
→ dokumentera ändringen i release‑notes

Detta förhindrar att en förändring av DTO:erna oväntat bryter UI eller andra tjänster.

***

### 11.9.5 Versionering per tjänst (semver + VERSION‑fil)

Varje tjänst har en egen:

    VERSION

som följer Semver:

    MAJOR.MINOR.PATCH

CI säkerställer:

*   att VERSION‑filen **ändras i PR** om OpenAPI har ändrats
*   att MAJOR‑bump görs när API görs inkompatibelt
*   att MINOR‑bump görs vid nya fält/endpoints
*   att PATCH används när kontraktet är oförändrat

Detta är en **stark garanti** för att API‑kontraktet är stabilt för alla konsumenter.

***

### 11.9.6 DTO‑validering i backend (runtime)

Time‑, Auth‑ och Group‑service använder automatiska validators baserade på JSON‑schema från OpenAPI.

Konsekvens:

*   alla inkommande requests **valideras**
*   backend returnerar **validerade responses**
*   inget kan smygas in som inte matchar DTO:n

Exempel på verktyg:

*   `league/openapi-psr7-validator`
*   `cebe/php-openapi`
*   JSON-schema validerare

Detta är sista försvarslinjen om något trots allt fel hamnar i API‑definitionen.

***

### 11.9.7 Mock‑server för frontendutveckling

Frontend teamet (eller du själv) använder:

*   `prism mock`
*   `openapi-mock-server`

för att generera en **simulerad backend** direkt från DTO:erna och OpenAPI.

På så vis:

*   kan UI utvecklas innan backend är färdig
*   backend och frontend arbetar mot samma datamodell
*   alla ändringar i DTO:er syns omedelbart i mock‑servern

Detta glider rakt in i kontraktstester och CI‑flöden.

***

### 11.9.8 Sammanfattning

Systemet använder en kombination av:

*   **OpenAPI som kontraktets enda källa**
*   **autogenererade klienter i frontend**
*   **granskning av DTO‑ändringar i CI**
*   **breaking change‑detektering (openapi-diff)**
*   **semver‑versionering (VERSION‑fil per tjänst)**
*   **runtime‑validering i backend**
*   **mock‑server för UI-utveckling**

Tillsammans garanterar detta att:

*   API‑kontrakt är stabila
*   DTO‑er aldrig förändras av misstag
*   backend och frontend alltid är kompatibla
*   breaking changes fångas innan deploy
*   API‑nivån kan utvecklas kontrollerat över tid

Detta är en modern, industristandardiserad och mycket robust mekanism för att **låsa API‑kontrakt** i en mikrotjänstarkitektur.

***

## 11.10 Sammanfattning

OpenAPI tillför följande till plattformen:

*   **Kontraktsstyrd utveckling**
*   **Typgenererade klienter** i alla frontends
*   **Dokumentation** utan manuell ansträngning
*   **Mock‑servrar** för parallell utveckling
*   **Validering** via CI/CD
*   **Byggblock för stabil versionering**

Det är ett modernt, hållbart och kraftfullt sätt att hålla API:erna konsistenta i en snabbt växande mikrotjänstplattform.

***

# 12. Versionshantering (Semver & Mikrotjänster)

Detta kapitel beskriver hur versioner hanteras i plattformens mikrotjänster och UI‑applikationer. Varje tjänst versioneras **oberoende**, enligt principerna för Semantic Versioning (semver), och versionerna kontrolleras och upprätthålls genom CI/CD‑processen.

Målet är att säkerställa:

*   tydliga, spårbara versioner för varje tjänst
*   möjlighet att rulla tillbaka enstaka komponenter
*   stabila API‑kontrakt
*   förutsägbar release‑cykel
*   kontroll i både lokal utveckling och CI/CD

***

## 12.1 Semantisk versionering (Semver)

Varje tjänst och varje frontendprojekt följer **Semantic Versioning**:

    MAJOR.MINOR.PATCH

Betydelse:

*   **MAJOR**  
    Brytande ändringar (breaking changes) som kan kräva kodändringar i klienter.

*   **MINOR**  
    Nya funktioner som är bakåtkompatibla.

*   **PATCH**  
    Bugfixar eller ändringar som inte påverkar externt beteende.

Exempel:

*   `2.0.0` – större omarbetning, bryter kompatibilitet
*   `1.3.0` – ny funktion men bakåtkompatibel
*   `1.3.4` – mindre buggfix

***

## 12.2 Versionering per mikrotjänst

Varje tjänst har en egen, fristående version.  
Det betyder:

*   Auth-service har sin version
*   Time-service har sin version
*   Results-service har sin version
*   Competition-service har sin version
*   Gateway och UI:er har sina versioner

På så sätt kan en tjänst uppdateras och deployas utan att påverka övriga tjänster.

***

## 12.3 VERSION‑fil per tjänst

Varje mikrotjänst innehåller en fil:

    VERSION

med endast:

    1.4.2

Denna fil:

*   är den **ensamma källan för sanningen** om versionen
*   används av CI/CD för att tagga Docker‑images
*   ingår i repo-versioneringen
*   måste uppdateras vid relevanta kodändringar

Fördelar:

*   ingen förvirring om versioner
*   lätt att inspektera manuellt
*   lätt att validera automatiskt

***

## 12.4 Regler för versionbumpning

### 1) PATCH-bump krävs när:

*   buggfixar
*   justeringar av validering
*   interna kodförbättringar utan beteendeförändring
*   prestandaoptimeringar

### 2) MINOR-bump krävs när:

*   nya endpoints läggs till
*   nya fält i API‑respons läggs till
*   affärslogik utökas utan att bryta bakåtkompatibilitet
*   UI-funktionalitet byggs ut utan att bryta tidigare API

### 3) MAJOR-bump krävs när:

*   API‑ändringar som inte är bakåtkompatibla
*   endpoints tas bort
*   fält tas bort eller ändras på ett inkompatibelt sätt
*   semantiken ändras så att klienter måste uppdateras
*   stora strukturella förändringar i tjänsten
*   ändring av autentiseringsmetod eller säkerhetsflöde

***

## 12.5 CI‑kontroll av versionering

CI/CD säkerställer att versionen är korrekt genom följande kontroller:

### 1) VERSION måste ändras om kod ändrats

CI stoppar bygget om:

*   filer i `src/` eller `actions/` ändrats
*   inga ändringar gjorts i `VERSION`‑filen

### 2) OpenAPI‑diff (valfritt men rekommenderat)

Om OpenAPI-specifikationen ändras kan CI upptäcka:

*   breaking changes → kräver MAJOR
*   additions → MINOR
*   inga schemaändringar → PATCH

Detta gör API‑versioneringen förutsägbar och säker.

### 3) Versionsnummer används vid Docker‑taggning

CI genererar taggar:

    <service>:1.4.2
    <service>:1.4
    <service>:1
    <service>:latest

### 4) Git-taggar skapas automatiskt vid release

Om du vill (valfritt):

    git tag auth-service-v1.4.2

***

## 12.6 Versionering av UI‑projekt

Varje frontend:

*   PWA Tränare
*   PWA Aktiva
*   Main UI

har sin egen versionering enligt samma semver-regler.

Detta är viktigt eftersom UI:er och backend kan utvecklas oberoende.

UI-versioneringen styr bl.a.:

*   deployment
*   cache-busting
*   kompatibilitet mellan UI och gateway
*   release‑notes

***

## 12.7 Release‑flöde

Ett typiskt releaseflöde i CI:

1.  PR godkänns
2.  CI kontrollerar:
    *   tester
    *   kodstil
    *   mutationstester
    *   openapi-linter
    *   versionbump
3.  Docker‑image byggs med version
4.  Image pushas till registry
5.  Deploy görs med `docker compose pull && docker compose up -d`
6.  Phinx‑migreringar körs
7.  En release med versionnumret skapas

***

## 12.8 Rollback-strategi

Eftersom tjänster är helt fristående:

*   Rollback görs genom att deploya en tidigare Docker‑tagg, t.ex.:

<!---->

    auth-service:1.3.2

*   Inga andra tjänster påverkas
*   Detta gör systemet mycket robust i produktion

***

## 12.9 Sammanfattning

Versioneringsstrategin säkerställer:

*   Konsistenta och pålitliga releaser
*   Förutsägbara API‑ändringar
*   Tydlig separation mellan tjänster
*   Automatisk validering via CI
*   Snabb rollback vid behov
*   Enkel, transparent hantering för utvecklare

Semver + VERSION‑fil + CI‑kontroll → **ett hållbart och lättförståeligt versioneringssystem för en mikrotjänstplattform.**

***

# 13. Roadmap

| Etapp  | Leverans                            |
| ------ | ----------------------------------- |
| **E1** | Auth-service + Gateway + magic‑link |
| **E2** | Main UI/Time-service/Group‑service + PWA Tränare |
| **E3** | Main UI/Results-service            |
| **E4** | Main UI/Statistik + PWA Aktiva     |
| **E5** | Competition-service                |


***

# 14. Release‑processen

Detta kapitel beskriver hur releaser hanteras i plattformen, från lokal utveckling till produktion. Processen säkerställer att alla tjänster och UI‑applikationer versioneras korrekt, testas, byggs och deployas på ett enhetligt och förutsägbart sätt — med minimal risk och full spårbarhet.

Målet är att:

*   alla mikrotjänster ska kunna deployas **oberoende** av varandra
*   alla UI‑projekt ska kunna releasas **utan påverkan på backend**
*   CI/CD ska garantera att kvaliteten är konsistent
*   rollback ska vara **enkel och snabb**
*   versioner ska vara **spårbara och förutsägbara**
*   inga brytande ändringar ska nå produktion av misstag

***

## 14.1 Översikt

En release består av:

1.  Kodförändring i en mikrotjänst eller UI
2.  Lokal validering (test + kodkvalitet)
3.  Pull Request till huvudbranch
4.  CI‑körning (test, analys, bygg, versionkontroll)
5.  Skapande av artefakt (Docker‑image eller UI‑bundle)
6.  Taggning med version (från `VERSION`‑filen)
7.  Deploy till produktion
8.  Körning av databasmigreringar (vid backend‑release)
9.  Verifiering i produktion

Varje tjänst och UI följer exakt samma stegstruktur, men har sina egna pipelines.

***

## 14.2 Lokal utveckling (pre‑release)

Vid lokal utveckling ansvarar utvecklaren för att:

### ✔ Köra alla testverktyg lokalt

*   PHPUnit
*   PHPStan
*   PHPCS
*   PHP‑CS‑Fixer
*   Infection
*   Rector (dry‑run)

### ✔ Låta GrumPHP verifiera kommittar

GrumPHP förhindrar "defekta" commits:

*   felaktig kodstil
*   brutna tester
*   statiska analysfel
*   onödiga diffar
*   osv.

### ✔ Uppdatera `VERSION`‑filen

Utvecklaren bump:ar versionen enligt:

*   PATCH – småfixar
*   MINOR – nya funktioner
*   MAJOR – breaking changes

Versionen är **obligatorisk att uppdatera** när kod ändras.

***

## 14.3 Pull Request‑processen

När en PR skapas kör CI:

1.  **PHPCS**
2.  **PHPStan**
3.  **PHPUnit + coverage**
4.  **Infection (mutation test)**
5.  **Rector (dry‑run)**
6.  **GrumPHP (CI‑version)**
7.  **OpenAPI‑linter** (om tjänsten använder OpenAPI)
8.  Kontroll att `VERSION` har bumpats
9.  Kontroll att eventuella OpenAPI‑ändringar matchar semver

Pull Request får **inte** mergeras om något av ovanstående misslyckas.

***

## 14.4 Build‑fas (skapa artefakt)

### Backend

Varje mikrotjänst bygger:

*   en Docker‑image
*   taggad enligt `VERSION`‑filen, t.ex.:

<!---->

    auth-service:1.4.2
    auth-service:1.4
    auth-service:1
    auth-service:latest

### Frontend (UI och PWAs)

Bygger en kompilerad bundle:

    /dist
       index.html
       assets/*

…som sedan deployas som statiska filer via:

*   S3/Cloudflare Pages
*   Docker‑image
*   eller traditionell webserver

(Valfritt, beroende på vald hostingmodell.)

***

## 14.5 Release i CI

När PR är mergad till huvudbranch kör CI automatiskt ett releaseflöde:

1.  Läser `VERSION`‑filen
2.  Skapar en git‑tagg, t.ex.:

<!---->

    results-service-v1.7.0

3.  Bygger Docker‑image
4.  Pushar till registry (DockerHub, GitHub Registry eller privat registry)
5.  Genererar release‑notes (valfritt)
6.  Triggar deployment‑steget

***

## 14.6 Deployment till produktion

Efter att en artefakt byggts pushas den till produktion:

### Backend

På produktionsserver:

    docker compose pull
    docker compose up -d

Sedan körs migrations:

    docker compose exec <service> phinx migrate

Varje tjänst migrerar **endast sin egen databas**.

### Frontend

UI‑byggen ersätter tidigare version:

*   vid S3/Cloudflare: filupload + cache-busting
*   vid Docker: ny container startas
*   vid klassisk server: deployment av nya statiska filer

***

## 14.7 Efterkontroller (post‑release)

Efter deploy körs automatiskt:

*   hälso‑kontroller (gateway ping)
*   statuskoll av respektive tjänsts `/health` endpoint
*   logginspektion (Monolog)
*   databasstatus och migreringskontroll

Detta sker automatiskt men går också att köra manuellt vid behov.

***

## 14.8 Rollback

Eftersom varje tjänst har sin egen version räcker det med:

    docker compose stop results-service
    docker pull results-service:1.6.3
    docker compose up -d

Ingen annan tjänst påverkas.

UI‑rollback görs på motsvarande sätt:

*   deploya tidigare UI‑bundle
*   eller använda versionerad Docker‑tagg

Rollback ska ta **sekunder, inte minuter**.

***

## 14.9 Versionering av Gateway

Gateway har egen version, vanligtvis:

*   bumpas vid ändring av routing
*   *inte* vid ändringar av tjänsternas interna API
*   bumpas vid ändring av säkerhetspolicy
*   sällan MAJOR (endast vid stora omstruktureringar)

Gateway följer samma release-process som andra tjänster.

***

## 14.10 Sammanfattning av releaseprocessen

Releaser är:

*   **automatiserade**
*   **versionsstyrda**
*   **testade innan merge**
*   **byggda via CI**
*   **deployade via Docker Compose**
*   **fullt spårbara**
*   **rollback‑vänliga**
*   **oberoende per mikrotjänst**

Denna process gör plattformen stabil, skalbar och lätt att utveckla vidare — utan att hela systemet måste deployas samtidigt.

***

# 15. Risker & mitigering

Detta kapitel sammanfattar de primära tekniska riskerna i plattformens mikrotjänstarkitektur samt föreslagna åtgärder för att säkerställa stabil drift, robust API‑kontrakt, korrekt versionering och en trygg release‑process.

***

## 15.1 Risk: API‑kontrakt divergerar från implementation

Ett API kan brytas om backend uppdateras utan att motsvarande OpenAPI‑specifikation uppdateras, eller om frontend bygger mot felaktigt kontrakt.

**Mitigering:**

*   OpenAPI-specifikation per tjänst
*   CI: OpenAPI‑linter + schema‑validering
*   CI: openapi-diff för att upptäcka breaking changes
*   automatskapade TypeScript-klienter i UI
*   versionbump (MAJOR/MINOR) baserat på kontraktsförändringar

***

## 15.2 Risk: Felaktig semver‑versionering

En breaking change kan av misstag märkas som PATCH eller MINOR, vilket kan leda till att klienter kraschar i produktion.

**Mitigering:**

*   VERSION‑fil per tjänst
*   CI: kontroll att VERSION ändrats vid kodändring
*   CI: OpenAPI‑diff → krav på korrekt semver‑bump
*   tydliga guidelines i utvecklingsprocessen

***

## 15.3 Risk: Bristande isolering mellan tjänster

En tjänst kan råka anta kunskap om en annan tjänsts databas eller interna datamodell.

**Mitigering:**

*   strikt mikrotjänstgräns, ingen delad SQL
*   endast kommunikation via Gateway eller intern service‑to‑service HTTP
*   separata databaser och egna migrations

***

## 15.4 Risk: Historiska data ändras oavsiktligt

Resultat, namn eller licenser skulle kunna ändras vid import eller uppdateringar.

**Mitigering:**

*   snapshot‑lagring i results-db
*   versionerade tabeller: `*_history`
*   immutable-values i historiska posttyper
*   datavalidering vid importflöden

***

## 15.5 Risk: Långa PWA‑sessioner (säkerhetsrisk)

Refresh tokens i PWA:er kan missbrukas vid enhetstapp eller stöld.

**Mitigering:**

*   kortlivade access tokens, långlivade refresh tokens
*   revocation endpoint
*   device‑binding via token claims
*   TOTP/OTP för känsliga operationer i UI

***

## 15.6 Risk: Driftproblem i databaser

Kraschad databas eller bristande backup kan få stora följder.

**Mitigering:**

*   MariaDB körs som värdtjänst, ej container
*   nattliga backuper + restore‑test
*   monitoring + loggning
*   migrations körs kontrollerat via CI/CD

***

## 15.7 Risk: Dålig testkvalitet eller låg kodhälsa

Minsta fel i affärslogik kan få dominoeffekter mellan tjänster.

**Mitigering:**

*   PHPUnit tester för services och domän
*   PHPStan för typ‑ och logikfel
*   PHPCS + PHP‑CS‑Fixer för kodstil
*   Infection mutation testing
*   Rector för automatiserad kodmodernisering
*   GrumPHP pre‑commit‑kontroller

***

## 15.8 Risk: Otillräcklig release‑kontroll

Fel version kan deployas, eller deploy sker utan fullständig testning.

**Mitigering:**

*   release‑process enligt kapitel 16
*   Docker‑taggar bundna till versioner
*   obligatoriskt CI‑genomflöde innan merge
*   automationsregler för versionbump och taggning
*   enkel rollback via semver‑taggar

***

# 16. Beroenden / npm & composer (uppdaterad)

Detta kapitel listar alla centrala beroenden som används i backend och frontend, inklusive verktyg för testning, statisk analys, mutationstester, kodstil, OpenAPI, klientgenerering och releasekvalitet.

***

## 16.1 Composer‑beroenden (backend)

### Core frameworks & DI

*   `slim/slim`
*   `php-di/php-di`
*   `nyholm/psr7` eller `slim/psr7`

### Databashantering

*   `doctrine/dbal`
*   `robmorgan/phinx`

### Autentisering & säkerhet

*   `firebase/php-jwt`
*   `phpmailer/phpmailer` (magic‑link utskick, vid behov)

### Loggning

*   `monolog/monolog`

***

### Kodkvalitet & analys

*   `phpunit/phpunit` — enhetstester
*   `phpstan/phpstan` — statisk analys
*   `squizlabs/php_codesniffer` — PHPCS (kodstandard)
*   `friendsofphp/php-cs-fixer` — automatisk kodformatering
*   `infection/infection` — mutationstester
*   `rector/rector` — automatiserad modernisering
*   `phpro/grumphp` — pre‑commit‑hooks och kvalitetskontroller

Dessa verktyg används både lokalt och i CI/CD.

***

## 16.2 npm‑beroenden (UI / PWA)

### Core

*   `vue`
*   `vite`
*   `typescript` (för typning av stores och genererade klienter)
*   `pinia`
*   `axios`

### UI-ramverk

*   `vuetify` (endast Main UI)

***

## 16.3 OpenAPI‑relaterade npm‑beroenden

Dessa används för kontraktsstyrd utveckling, klientgenerering och dokumentation:

*   `openapi-typescript`  
    – genererar typade TS‑klienter från `openapi.yaml`
*   `@redocly/openapi-cli`  
    – linter + validering + best practices
*   `openapi-schema-validator`  
    – schematisk validering
*   `swagger-ui-dist` **eller** `redoc`  
    – för att generera dokumentation
*   `openapi-diff`  
    – upptäcker breaking changes vid versionshantering

Dessa verktyg är viktiga delar av CI/CD när OpenAPI används som stabilt kontrakt mellan frontend och backend.

***
Självklart — här kommer **Kapitel 17 omskrivet med korrekt rubriknivå**, dvs. **en rubrik på nivå 1 (`#`)**, inte nivå 2.

Alla underrubriker har justerats därefter (nivå 2 och 3).  
Inget annat i innehållet är ändrat — bara rubriknivåerna.

Detta är den korrekta slutliga versionen för din specifikation.

***

# 17. GitHub‑organisation – Struktur, repos, branchpolicy och PR‑rutiner

Detta kapitel beskriver den rekommenderade strukturen för GitHub‑organisationen som rymmer alla mikrotjänster, UI‑projekt och dokumentation för plattformen.  
Målet är att:

*   skapa en enhetlig struktur
*   säkerställa god kodkvalitet
*   möjliggöra CI/CD per tjänst
*   följa branschstandarder
*   ge en skalbar lösning för framtiden

***

## 17.1 Organisationens namn och syfte

En GitHub‑organisation samlar alla delar av plattformen under ett och samma “tak”.  
Organisationen används för:

*   versionshantering
*   CI/CD
*   teamhantering
*   Secrets/environments
*   PR‑policy
*   gemensamma templates
*   dokumentation

***

## 17.2 Repo‑struktur (namn och ansvar)

Alla repos följer konventionen:

    <service-name>-service

### Backend/mikrotjänster

*   `auth-service`
*   `time-service`
*   `group-service`
*   `gateway-service`

*(Framtida repos:)*

*   `results-service`
*   `competition-service`

### Frontend / PWA

*   `pwa-trainer`
*   `pwa-active`
*   `main-ui`

### Dokumentation / övrigt

*   `platform-docs`
*   `.github` (globala templates och workflows)
*   `infrastructure` 

***

## 17.3 Namngivningsprinciper

### För tjänster

*   `auth-service`
*   `time-service`
*   `group-service`
*   `gateway-service`

### För frontend

*   `pwa-trainer`
*   `pwa-active`
*   `main-ui`

### För dokumentation

*   `platform-docs`

Enhetlighet skapar förutsägbarhet för CI/CD och makes dev‑ops life easier.

***

## 17.4 Branch‑policy

Modell: **main + feature branches**

### Permanenta branches:

*   `main` – alltid stabil och deploybar

### Feature branches:

    feature/<kort-namn>
    fix/<kort-namn>
    refactor/<kort-namn>

### Skyddade branches:

*   `main`

**Krav för att få merge:**

*   minst 1 approved review
*   CI måste vara grön
*   VERSION‑fil måste vara uppdaterad vid OpenAPI‑ändring
*   inga direkt‑commits till main

***

## 17.5 Pull Request‑policy

### Krav:

*   PR måste vara kopplad till ett issue (valfritt men rekommenderat)
*   PR måste beskriva ändringen
*   CI körs automatiskt och måste vara grön:
    *   PHPUnit
    *   PHPStan
    *   PHPCS
    *   PHP‑CS‑Fixer (dry-run)
    *   Rector (dry-run)
    *   Infection (mutation testing)
    *   OpenAPI‑linter
    *   openapi-diff
*   Version måste bumpas vid API‑ändringar

### PR‑template (förslag):

    ## Summary
    Kort sammanfattning av ändringen.

    ## Type of change
    - [ ] Bug fix
    - [ ] New feature
    - [ ] Breaking change
    - [ ] Documentation

    ## API impact
    - [ ] DTOs changed
    - [ ] OpenAPI updated
    - [ ] version bump required

    ## Checklist
    - [ ] CI passes
    - [ ] Code formatted
    - [ ] Tests added/updated

***

## 17.6 CODEOWNERS

En `.github/CODEOWNERS` fil säkerställer att rätt personer godkänner PR:er.

Exempel:

    /auth-service/ @you
    /time-service/ @you
    /group-service/ @you
    /gateway-service/ @you

    /platform-docs/ @you

***

## 17.7 .github‑repo (centrala resurser)

Organisationen bör ha ett repo `.github` med:

    .github/
      workflows/
        php-ci.yml
        openapi-check.yml
        security.yml
      PULL_REQUEST_TEMPLATE.md
      ISSUE_TEMPLATE/
        bug_report.md
        feature_request.md
        documentation.md
      CODEOWNERS
      CONTRIBUTING.md

Dessa används automatiskt av alla repos i organisationen.

***

## 17.8 Secrets, environments och deploy‑policy

Organisationen använder GitHub Environments:

    dev
    stage
    prod

I varje environment lagras:

*   secrets (DB‑lösenord, JWT‑nycklar, API‑tokens)
*   godkännandesteg innan deploy
*   branch‑restriktioner

Exempel:

    prod:
      requires approval
      reviewers: @you

***

## 17.9 Release‑policy per repo

Varje tjänst har en `VERSION`‑fil och följer semver enligt kapitel 14.

**CI‑regler:**

*   MAJOR bump vid breaking API
*   MINOR bump vid nya endpoints/fält
*   PATCH vid fixar
*   GitHub Release skapas
*   Docker‑image taggas:
    *   `<service>:X.Y.Z`
    *   `<service>:X.Y`
    *   `<service>:X`
    *   `<service>:latest`

***

## 17.10 Sammanfattning

GitHub‑organisationen skapar:

*   professionell struktur för mikrotjänster
*   standardiserat arbetssätt
*   enklare CI/CD
*   tydlig separation per tjänst
*   ren versionshantering
*   bra för framtida expansion och fler utvecklare

Detta kapitel utgör best practice för GitHub‑hantering i denna plattform.

***
