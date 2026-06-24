# OpenAPI CI

OpenAPI CI ansvarar för att verifiera och skydda plattformens API‑kontrakt.

CI‑flödet säkerställer att förändringar i API:t är:

- syntaktiskt korrekta
- kompatibla med tidigare versioner
- korrekt versionshanterade

OpenAPI betraktas som en förstaklass‑artefakt i systemet och används av både backend och frontend.

---

## Aktivering

OpenAPI CI körs i repositories som deltar i API‑kontraktet:

- backend service‑repos (via `.ci/php-service`)
- frontend‑repos (via `.ci/frontend`)

Repositories utan någon av dessa flaggor:

- kör inte OpenAPI‑check
- behandlas som template eller stöd‑repos

---

## Funktionalitet

OpenAPI CI består av följande kontroller:

### 1. Syntaxvalidering

- verifierar att `openapi.yaml` är korrekt formaterad
- säkerställer att filen kan parsas

---

### 2. Schema‑validering

- kontrollerar att OpenAPI‑specifikationen följer standard
- verifierar endpoints, request/response‑schemas och typer

---

### 3. Breaking change detection (Pull Requests)

Vid pull requests jämförs aktuell OpenAPI med `main`:

- identifierar inkompatibla ändringar
- blockerar merge om breaking changes upptäcks

Exempel på breaking changes:

- borttagna endpoints
- ändrade response‑typer
- borttagna eller ändrade obligatoriska fält

---

### 4. VERSION‑kontroll

Om `openapi.yaml` ändras krävs:

- uppdatering av VERSION‑filen

Syfte:

- säkerställa att API‑förändringar versionshanteras
- möjliggöra spårbarhet mellan API och deploy

---

### 5. Kontraktstester

Backend‑tjänster ska innehålla integrationstester som verifierar att implementationens responses överensstämmer med den publicerade OpenAPI‑specifikationen.

Kontroller inkluderar:

- statuskoder
- response‑schema
- felmodeller
- obligatoriska och valfria fält

Syftet är att säkerställa att implementation och dokumentation inte divergerar över tid.

---

## Relation till Backend CI

Backend definierar och publicerar OpenAPI‑kontraktet.

OpenAPI CI säkerställer att:

- kontraktet är korrekt
- förändringar inte bryter existerande beteende
- versionering följs

---

## Relation till Frontend CI

Frontend använder OpenAPI‑specifikationen för:

- generering av TypeScript‑definitioner
- typkontroll av API‑anrop

OpenAPI CI säkerställer att:

- frontend alltid är kompatibel med API‑kontraktet
- kontraktsbrott upptäcks innan deployment

---

## Syfte

OpenAPI CI säkerställer att:

- API‑kontraktet är korrekt och validerat
- breaking changes upptäcks tidigt
- versionering följs konsekvent
- backend och frontend förblir synkroniserade
- implementationen överensstämmer med den publicerade OpenAPI‑specifikationen

---

## Begränsningar

OpenAPI CI omfattar inte:

- affärslogik
- routing
- domänbeteende
- funktionell korrekthet utanför det definierade API‑kontraktet

OpenAPI CI verifierar både kontraktet och att implementerade endpoints följer kontraktet, men avgör inte om affärslogiken bakom endpointen är korrekt.

---

## Sammanfattning

OpenAPI CI är ett centralt kvalitetssäkringssteg i plattformen:

- backend definierar kontraktet
- frontend konsumerar kontraktet
- OpenAPI CI skyddar kontraktet

Detta möjliggör:

- trygg refactoring
- säker utveckling i parallella repos
- tydlig versionshantering av API:t