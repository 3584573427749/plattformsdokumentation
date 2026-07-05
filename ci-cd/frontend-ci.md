# Frontend CI

Frontend CI ansvarar för att verifiera kvalitet, byggbarhet och API-kompatibilitet för plattformens frontend-applikationer. [1](https://alandsgymnasium-my.sharepoint.com/personal/kjellh_gymnasium_ax/Documents/Microsoft%20Copilot%20Chat-filer/frontend-ci.md)

Frontend CI används för:

- Vue-baserade PWA-applikationer
- Vue-baserade webbgränssnitt (Admin UI och andra webbklienter)

## Aktivering

Frontend CI körs endast i repositories som innehåller flaggfilen:

```text
.ci/frontend
```

Denna fil används för att:

- indikera att repot är ett aktivt frontend-repo
- aktivera frontend-relaterade CI-flöden

Repositories utan denna fil:

- behandlas som template- eller stöd-repositories
- kör inte frontend CI (workflows skippar automatiskt) [1](https://alandsgymnasium-my.sharepoint.com/personal/kjellh_gymnasium_ax/Documents/Microsoft%20Copilot%20Chat-filer/frontend-ci.md)

## Frontend CI – Funktionalitet

Frontend CI ansvarar för att verifiera att frontend-applikationen:

- kan byggas korrekt
- följer kodstandard
- fungerar enligt tester
- har giltiga OpenAPI-definitioner när OpenAPI används

### Innehåll

Frontend CI innehåller följande steg:

- installation av beroenden (npm)
- generering av TypeScript-typer från OpenAPI (om `openapi.yaml` finns)
- linting (ESLint)
- enhetstester (Vitest)
- byggning av produktionsartefakt (`npm run build`)

## OpenAPI Type Generation

Om `openapi.yaml` finns i repot genereras TypeScript-typer från API-kontraktet.

Generering sker via:

```bash
npm run generate-types
```

Resultatet sparas i:

```text
src/generated/types.ts
```

Typerna används av:

- API-services
- Pinia stores
- Vue-komponenter

Detta säkerställer att frontend är typmässigt synkroniserat med backend-kontraktet.

## Versionering

Frontend CI hanterar inte versionskontroll.

VERSION-filen verifieras endast i release-pipelinen när en release-tagg skapas.

Vanliga Pull Requests och merge till main kräver ingen versionsuppdatering.

## OpenAPI Validation

Frontend deltar i plattformens OpenAPI-validering via separat workflow.

### Innehåll

- validering av OpenAPI-syntax
- schema-validering

### Syfte

- säkerställa att OpenAPI-specifikationen är giltig
- säkerställa att frontend kan generera typer från kontraktet
- upptäcka felaktiga eller ogiltiga API-definitioner tidigt

## Security Scan

Frontend-repositories inkluderas i plattformens Security Scan-workflow.

### Innehåll

```bash
npm audit --audit-level=high
```

### Syfte

- identifiera kända sårbarheter med hög eller kritisk allvarlighetsgrad i frontend-beroenden

## Syfte

Frontend CI säkerställer att:

- frontend kan byggas utan fel
- kodstandard följs
- tester passerar
- API-integration är korrekt typad
- OpenAPI-specifikationer är giltiga

## Begränsningar

Frontend CI omfattar inte:

- Docker-build
- release-processen
- deployment
- runtime-konfiguration
- CDN- eller hostingstrategier

Dessa hanteras av separata release- och deploy-workflows.

## Sammanfattning

Frontend CI är uppdelad i flera specialiserade kontroller:

- bygg och tester verifieras i Frontend CI
- API-kontrakt verifieras i OpenAPI Validation
- beroenden kontrolleras i Security Scan

Denna uppdelning ger:

- tydlig ansvarsfördelning
- snabb och fokuserad CI
- konsekvent beteende mellan frontend och backend
- stabila och reproducerbara releaser