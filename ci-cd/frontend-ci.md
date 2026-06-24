# Frontend CI

Frontend CI ansvarar för att verifiera kvalitet, byggbarhet och API‑kompatibilitet för plattformens frontend‑applikationer.

Frontend CI används för:

- Vue-baserade PWA-appar
- Vue-baserade webbgränssnitt (web UI)

---

## Aktivering

Frontend CI körs endast i repositories som innehåller flaggfilen:

.ci/frontend

Denna fil används för att:

- indikera att repot är ett aktivt frontend‑repo
- aktivera frontend‑relaterade CI‑flöden

Repositories utan denna fil:

- behandlas som template‑ eller stöd‑repos
- kör inte frontend CI (workflows skippar automatiskt)

---

## Frontend CI – Funktionalitet

Frontend CI ansvarar för att verifiera att frontend‑applikationen:

- kan byggas korrekt
- följer kodstandard
- fungerar enligt tester

### Innehåll

Frontend CI innehåller följande steg:

- installation av beroenden (npm)
- generering av TypeScript‑definitioner från OpenAPI (om `openapi.yaml` finns)
- linting (ESLint)
- enhetstester (t.ex. Vitest/Jest)
- byggning av produktionsartifact (npm run build)

### OpenAPI type generation

Om `openapi.yaml` finns i repot:

- genereras `.d.ts`-filer automatiskt
- dessa används för typning av API‑anrop i frontend

Detta säkerställer att frontend är typmässigt synkroniserat med backend‑kontraktet.

---

## VERSION‑policy

Frontend CI innehåller en kontroll som säkerställer:

- att VERSION‑filen uppdateras när källkod ändras

Detta gäller när:

- filer i `src/` ändras

Syftet är att:

- koppla kodändringar till versionshantering
- säkerställa att nya builds kan spåras

---

## OpenAPI Validation

Frontend deltar i plattformens OpenAPI‑validering via separat workflow: openapi-check.yml.

### Innehåll

- validering av OpenAPI‑syntax
- schema‑kontroll
- verification av breaking changes

### Syfte

- säkerställa att frontend är kompatibelt med API‑kontraktet
- upptäcka kontraktsbrott tidigt

---

## Security Scan

Frontend‑repos inkluderas i Security scan workflow.

### Innehåll

- npm audit (dependency scanning)

### Syfte

- identifiera kända sårbarheter i frontend‑beroenden

---

## Syfte

Frontend CI säkerställer att:

- frontend kan byggas utan fel
- kodstandard följs
- tester passerar
- API‑integration är korrekt typad
- förändringar versionshanteras korrekt

---

## Begränsningar

Frontend CI omfattar inte:

- Docker build eller deployment
- runtime‑konfiguration
- CDN eller hosting‑strategier

Dessa hanteras separat i deployment‑dokumentation.

---

## Sammanfattning

Frontend CI är uppdelad i flera specialiserade kontroller:

- bygg och tester verifieras i Frontend CI
- API‑kontrakt verifieras i OpenAPI validation
- beroenden kontrolleras i Security scan

Denna uppdelning ger:

- tydlig ansvarsfördelning
- snabb och fokuserad CI
- konsekvent beteende mellan frontend och backend
``
