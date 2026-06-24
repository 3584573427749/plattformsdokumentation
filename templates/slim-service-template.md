# Slim Service Template

Slim Service Template är grundmallen för backend‑tjänster inom plattformen.

Syftet med mallen är att ge alla backend‑tjänster samma tekniska grund, projektstruktur och integrationspunkter mot plattformens gemensamma CI/CD‑processer.

---

## Ingående komponenter

Mallen innehåller färdig konfiguration för:

- Slim 4
- PHP-DI
- Doctrine DBAL
- PHPUnit
- PHPStan
- PHP-CS-Fixer
- PHPCS
- Rector
- Phinx
- Docker
- OpenAPI

---

## Projektstruktur

Mallen innehåller en fördefinierad projektstruktur för:

- HTTP-lager
- applikationslager
- domänlager
- infrastruktur
- tester

Den exakta strukturen kan utvecklas över tid men ska följa plattformens backend‑riktlinjer.

---

## OpenAPI

Varje backend‑tjänst ska definiera sitt publika API via:

```text
openapi.yaml
```

Mallen innehåller stöd för:

- OpenAPI‑baserad utveckling
- kontraktsvalidering
- integration med OpenAPI CI

---

## Databas

Mallen innehåller:

- Doctrine DBAL
- databasabstraktion
- stöd för databasmigreringar via Phinx

Varje tjänst ansvarar för sin egen databas och sina egna migreringar.

---

## Testning

Mallen innehåller stöd för:

- enhetstester
- integrationstester
- OpenAPI‑baserade kontraktstester

Teststrategi och kodstandard beskrivs i separata dokument.

---

## CI/CD

Mallen är förberedd för plattformens gemensamma CI/CD‑flöden.

Aktivering sker genom flaggfiler i `.ci/`.

Exempel:

```text
.ci/php-service
```

Detta aktiverar backend‑relaterade kvalitetskontroller såsom:

- tester
- statisk analys
- OpenAPI‑validering
- säkerhetskontroller

---

## Versionering

Mallen innehåller en VERSION‑fil.

VERSION är den enda källan till sanningen för tjänstens version och används av plattformens releaseprocess.

Versionering följer Semantic Versioning (SemVer).

---

## Docker

Mallen innehåller en färdig Docker‑konfiguration för lokal utveckling och releasebyggnation.

Backend‑tjänster distribueras som Docker‑images via GitHub Container Registry (GHCR).

---

## Syfte

Målet med Slim Service Template är att:

- minska uppstartstiden för nya tjänster
- säkerställa teknisk enhetlighet
- förenkla drift och underhåll
- möjliggöra gemensamma CI/CD‑flöden
- ge en konsekvent utvecklarupplevelse inom hela plattformen