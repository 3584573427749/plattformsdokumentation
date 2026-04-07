# Plattformdokumentation

Detta repository innehåller all centralt teknisk dokumentation för idrottsplattformen. Här finns arkitektur, API‑kontrakt, CI/CD‑flöden, PWA‑strategi och utvecklarriktlinjer.

## Innehåll
- **architecture/** – systemöversikt, mikrotjänster, frontendmodell, tokenflow
- **api-contracts/** – OpenAPI‑policy, generation, versionering, breaking changes
- **templates/** – dokumentation för Slim, Web UI och PWA templates
- **infrastructure/** – docker-compose, traefik, deployment
- **ci-cd/** – backend-, frontend- och OpenAPI‑pipeline
- **development/** – riktlinjer, kodstil, branching, contributing
- **roadmap/** – milstolpar, releaser, faser

## Mikrotjänstarkitektur
Plattformen består av:
- Auth Service
- Gateway Service
- Time Service
- Group Service
- Results Service
- Entries Service

Alla tjänster använder:
Slim 4  
 PHP‑DI  
 Doctrine DBAL  
 OpenAPI‑kontrakt  
 CI/CD med GitHub Actions

## Frontend
Frontend består av:
- **Admin UI (Vuetify)**
- **Web UI (Vuetify)**
- **Entries UI (Vuetify)**
- **PWA Tränare**
- **PWA Aktiva**

Alla UI bygger på:
 Vue 3  
 Vite  
 typade DTO:er genererade från OpenAPI  
 Docker‑deployment  

## Autentisering
Systemet använder:
- Passwordless login (magic link eller one time passwords)
- JWT access tokens
- httpOnly refresh tokens
- Access‑token rotation per svar (header‑baserad)

## OpenAPI
All utveckling är kontraktsstyrd.
- Varje tjänst har en egen `openapi.yaml`
- Frontend genererar typdefinitioner automatiskt
- CI validerar kontrakten
- Breaking changes blockeras automatiskt

## Infrastruktur
Stacken består av:
- Docker Compose
- Traefik v3
- MariaDB
- Interna nätverk
- Versionering & deployments via GHCR

## Contributing
Se `development/contributing.md`.

## Roadmap
Se `roadmap/phases.md`.
