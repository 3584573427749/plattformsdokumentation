# Backend CI

Backend CI är den del av plattformens Continuous Integration (CI) som ansvarar för att verifiera funktionalitet, kodkvalitet och API‑kontrakt för backend‑tjänster.

Backend CI är inte ett enskilt workflow, utan består av flera samverkande CI‑flöden med tydligt ansvar:

- Backend CI (kod + tester)
- OpenAPI validation (API‑kontrakt)
- Security scan (beroendesäkerhet)

Docker build, release och deployment hanteras separat och ingår inte i denna del.

---

## Aktivering

Backend CI körs endast för repositories som identifieras som backend service‑repos.

Detta sker via en explicit flaggfil:

```text
.ci/php-service
````

Denna fil används för att:

* aktivera backend‑relaterade CI‑flöden
* indikera att repot innehåller en körbar PHP‑tjänst

Repositories utan denna fil:

* behandlas som template‑ eller stöd‑repos
* kör inte backend‑CI (workflows skippar automatiskt)

***

## Backend CI – Funktionalitet

Backend CI ansvarar för att verifiera kod, tester och grundläggande kodkvalitet.

### Innehåll

Backend CI innehåller följande steg:

* installation av beroenden via Composer
* körning av PHPUnit
* statisk analys via PHPStan
* kodstandardkontroll via PHPCS
* formatkontroll via PHP‑CS‑Fixer (dry‑run)
* kontroll av moderniseringsregler via Rector (dry‑run)

Beroende på tjänstens mognadsgrad kan även mutationstester via Infection ingå i CI‑kedjan.

### Syfte

Backend CI säkerställer att:

* koden är korrekt och testbar
* inga regressionsfel introduceras
* kodstandarder följs
* statiska analysproblem upptäcks tidigt
* kodbasen kan moderniseras kontrollerat över tid

***

## OpenAPI Validation

Backend‑tjänster definierar sina API‑kontrakt via OpenAPI.

Dessa kontrakt verifieras i ett separat workflow.

### Körs i

* backend service‑repos

### Innehåll

* validering av OpenAPI‑syntax
* schema‑kontroll
* kontroll av breaking changes vid pull requests
* krav på VERSION‑uppdatering vid ändringar i API‑kontrakt
* kontraktstester som verifierar att implementationens responses överensstämmer med OpenAPI‑specifikationen

### Syfte

* säkerställa att API‑kontrakt är giltiga
* verifiera att implementationen följer den publicerade OpenAPI‑specifikationen
* förhindra breaking changes utan korrekt versionshantering
* garantera kompatibilitet mellan backend och frontend
* säkerställa att OpenAPI är den enda källan till sanningen för API‑kontraktet

***

## Security Scan

Security scan kontrollerar beroenden i backend‑tjänster.

### Innehåll

* Composer audit (PHP‑beroenden)

### Syfte

* identifiera kända sårbarheter i dependencies
* ge tidig varning om säkerhetsproblem

Security scan körs automatiskt men påverkar inte alltid build‑status (informerande snarare än blockerande beroende på konfiguration).

***

## Utökad kvalitetssäkring

Utöver den grundläggande CI‑kedjan kan följande verktyg användas.

### Infection (mutationstester)

Infection används för att mäta testernas kvalitet genom mutationstestning.

Syftet är att säkerställa att testerna inte bara täcker kod utan även upptäcker beteendeförändringar.

Används typiskt:

* periodiskt i CI
* vid större refactoring
* vid kvalitetssäkring inför release

### Rector

Rector används för att modernisera och refaktorisera PHP‑kod.

* uppgraderar kod till nyare PHP‑versioner
* tillämpar best practices automatiskt
* reducerar teknisk skuld

Används typiskt:

* vid versionsuppgraderingar
* vid refactoring
* som dry‑run i CI
* manuellt vid större moderniseringar

***

## Begränsningar

Backend CI omfattar inte:

* Docker build eller release
* publicering av container‑images
* deployment till server
* runtime‑miljö eller driftövervakning

Dessa ansvar hanteras i separat dokumentation för release och deployment.

***

## Sammanfattning

Backend CI är uppdelad i flera specialiserade workflows:

* kod och tester verifieras i Backend CI
* API‑kontrakt verifieras i OpenAPI validation
* beroenden kontrolleras i Security scan

Tillsammans säkerställer dessa flöden att:

* kodkvaliteten är hög
* API‑kontrakten förblir korrekta
* implementation och dokumentation inte divergerar
* frontend och backend kan utvecklas parallellt utan integrationsproblem

Denna uppdelning ger:

* tydlig ansvarsfördelning
* snabbare och mer fokuserad CI
* möjlighet att utveckla varje del oberoende
* hög tillit till att releaser följer plattformens definierade kontrakt