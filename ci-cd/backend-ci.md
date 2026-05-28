# Backend CI

Backend CI är den del av plattformens Continuous Integration (CI) som ansvarar för att verifiera funktionalitet, kodkvalitet och kontrakt för backend‑tjänster.

Backend CI är inte ett enskilt workflow, utan består av flera samverkande CI‑flöden med tydligt ansvar:

- Backend CI (kod + tester)
- OpenAPI validation (API-kontrakt)
- Security scan (beroendesäkerhet)

Docker build och deployment hanteras separat och ingår inte i denna del.

---

## Aktivering

Backend CI körs endast för repositories som identifieras som backend service‑repos.

Detta sker via en explicit flaggfil:

.ci/php-service

Denna fil används för att:

- aktivera backend-relaterade CI‑flöden
- indikera att repot innehåller en körbar PHP‑tjänst

Repositories utan denna fil:

- behandlas som template‑ eller stöd‑repos
- kör inte backend‑CI (workflows skippar automatiskt)

---

## Backend CI – Funktionalitet

Backend CI ansvarar för att verifiera kod och tester.

### Innehåll

Backend CI innehåller följande steg:

- installation av beroenden via Composer
- körning av PHPUnit 
- statisk analys via PHPStan
- kodstandardkontroll (t.ex. PHP-CS-Fixer)

### Syfte

Backend CI säkerställer att:

- koden är korrekt och testbar
- inga regressionsfel introduceras
- kodstandarder följs
- statiska analysproblem upptäcks tidigt

---

## OpenAPI Validation

Backend‑tjänster definierar sina API-kontrakt via OpenAPI.

Dessa kontrakt verifieras i ett separat workflow: openapi-check.yml.

### Körs i

- backend service‑repos

### Innehåll

- validering av OpenAPI‑syntax
- schema‑kontroll
- kontroll av breaking changes vid pull requests
- krav på VERSION‑uppdatering vid ändringar i API-kontrakt

### Syfte

- säkerställa att API-kontrakt är giltiga
- förhindra breaking changes utan versionshantering
- garantera kompatibilitet mellan backend och frontend

---

## Security Scan

Security scan kontrollerar beroenden i backend‑tjänster.

### Innehåll

- Composer audit (PHP beroenden)

### Syfte

- identifiera kända sårbarheter i dependencies
- ge tidig varning om säkerhetsproblem

Security scan körs automatiskt men påverkar inte alltid build‑status (informerande snarare än blockerande beroende på konfiguration).

---

## Avancerad kodkvalitet (valfritt)

Följande verktyg ingår inte i standard Backend CI, men kan användas vid behov:

### Rector (automatisk refactoring)

Rector används för att modernisera och refaktorisera PHP‑kod.

- uppgraderar kod till nyare PHP‑versioner
- tillämpar best practices automatiskt

Används typiskt:

- vid versionsuppgraderingar
- vid refactoring
- manuellt eller i separata pipelines

---

## Begränsningar

Backend CI omfattar inte:

- Docker build eller release
- deployment till server
- runtime‑miljö eller driftövervakning

Dessa ansvar hanteras i separat dokumentation för deployment.

---

## Sammanfattning

Backend CI är uppdelad i flera specialiserade workflows:

- kod och tester verifieras i Backend CI
- API‑kontrakt verifieras i OpenAPI validation
- beroenden kontrolleras i Security scan

Denna uppdelning ger:

- tydlig ansvarsfördelning
- snabbare och mer fokuserad CI
- möjlighet att utveckla varje del oberoende
