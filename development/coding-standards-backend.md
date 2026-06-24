# Kodstandard – Backend

Detta dokument beskriver gemensamma kodstandarder och arkitekturprinciper för backend‑tjänster inom plattformen.

Syftet är att säkerställa:

* konsekvent kodstruktur
* hög läsbarhet
* enkel testbarhet
* tydlig ansvarsfördelning
* enhetlig utvecklarupplevelse mellan tjänster

---

## Generella principer

Backend‑tjänster ska:

* följa PSR‑12
* använda strikt typning (`declare(strict_types=1)`)
* använda beroendeinjektion
* undvika globalt tillstånd
* vara OpenAPI‑first

All kod ska vara lätt att testa, förstå och refaktorera.

---

## Arkitektur

Backend‑tjänster följer en lagerindelad arkitektur:

```text
HTTP
↓
Application
↓
Domain
↓
Infrastructure
````

Varje lager ansvarar för en tydligt avgränsad del av systemet.

***

## Actions

Actions representerar HTTP‑lagret.

Actions ansvarar för att:

* läsa request‑data
* hämta route‑parametrar
* validera inkommande data
* skapa kommandon eller query‑objekt
* anropa handlers
* returnera HTTP‑responses

Actions ska vara tunna.

Actions får inte innehålla affärslogik.

***

## Handlers

Handlers ansvarar för applikationslogik.

Handlers:

* koordinerar användningsfall
* använder repositories
* skapar eller returnerar DTO:er
* hanterar tjänstens arbetsflöden

Affärslogik ska placeras i handlers eller domänobjekt, inte i actions.

***

## Repositories

Repositories ansvarar för persistens.

Repositories:

* läser data
* sparar data
* uppdaterar data
* tar bort eller avaktiverar data

Repositories ska inte känna till HTTP eller OpenAPI.

***

## Domänmodell

Domänlagret ska innehålla:

* Entities
* Value Objects
* DTO:er
* Repository‑kontrakt
* Domänspecifika exceptions

Domänlagret får inte bero på Slim, HTTP eller databasteknik.

***

## Value Objects

Value Objects används för identifierare och specialiserade värden.

Exempel:

* UserId
* Email
* DateTimeValue

Validering ska i första hand ske i Value Objects när värdet representerar en domänregel.

***

## DTO

DTO:er används för dataöverföring mellan lager och mot API‑klienter.

DTO:er ska:

* vara immutabla där det är praktiskt möjligt
* följa OpenAPI‑kontraktet
* inte exponera interna implementationdetaljer

***

## Validering

Validering sker i flera steg:

### Request‑validering

Inkommande data valideras innan ett användningsfall exekveras.

Exempel:

* obligatoriska fält
* formatkontroll
* längdbegränsningar

### Domänvalidering

Domänspecifika regler hanteras i domänobjekt, Value Objects eller handlers.

***

## Felhantering

Exceptions ska hanteras centralt.

API:t ska returnera standardiserade felsvar:

```json
{
  "statusCode": 404,
  "error": {
    "type": "NotFoundException",
    "message": "User not found",
    "details": null
  }
}
```

Alla backend‑tjänster ska använda samma felmodell.

***

## OpenAPI

OpenAPI är den enda källan till sanningen för API‑kontraktet.

Alla publika endpoints ska dokumenteras i tjänstens OpenAPI‑specifikation.

API‑förändringar ska göras i OpenAPI innan implementationen uppdateras.

***

## Testning

Varje tjänst ska innehålla:

### Enhetstester

Verifierar:

* Value Objects
* Entities
* Validators
* Handlers
* Repositories

### Integrationstester

Verifierar:

* routing
* actions
* persistence
* felhantering
* OpenAPI‑kontrakt

### Kontraktstester

Kontraktstester ska verifiera att implementationens responses överensstämmer med OpenAPI‑specifikationen.

***

## Namngivning

Klasser namnges utifrån ansvar:

Exempel:

* GetUserAction
* CreateUserHandler
* UserRepository
* UpdateUserCommand
* UserDTO

Förkortningar ska undvikas om de inte är etablerade och självklara.

***

## Kodkvalitet

All kod ska passera:

* PHPUnit
* PHPStan
* PHPCS
* PHP‑CS‑Fixer

Kod som inte passerar CI ska inte mergas.

***

## Målsättning

Backend‑kod ska vara:

* enkel att förstå
* enkel att testa
* enkel att förändra
* konsekvent mellan tjänster
* oberoende av specifika implementationer där det är praktiskt möjligt
