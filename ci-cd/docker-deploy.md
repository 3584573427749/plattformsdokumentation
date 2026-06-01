# CI/CD – Deployment

Detta dokument beskriver hur plattformens applikationer byggs, publiceras och deployas.

Deployment är baserat på Docker och container images som byggs via GitHub Actions och publiceras till GitHub Container Registry (GHCR).

---

## Översikt

Deployment‑flödet består av tre steg:

1. **Build** – Docker image byggs i CI
2. **Publish** – image pushas till GHCR
3. **Deploy** – servern hämtar och startar ny version

Flödet är identiskt för:

- backend services (PHP)
- frontend‑applikationer (Vue PWA och web UI)

---

## Aktivering

Docker release körs endast i repositories som representerar körbara tjänster.

Detta styrs via `.ci/`‑katalogen:

- `.ci/dockerdeploy`
- 
Repositories utan denna flagga:

- betraktas som template‑ eller stöd‑repos
- bygger inte Docker images
- deployas inte

---

## Build & Publish

Docker images byggs och publiceras automatiskt via:

```

.github/workflows/docker-release.yml

```

### Trigger

- push till `main` branch

### Steg

- VERSION läses från VERSION‑fil
- Docker image byggs via Docker Buildx
- image pushas till GitHub Container Registry (GHCR)

---

## Image tagging

Varje build publicerar:

```
ghcr.io/<owner>/<repo>:<VERSION>
ghcr.io/<owner>/<repo>:latest

```

---

## Docker Image‑modell

All kod deployas som Docker containers.

### Backend

- PHP‑baserade mikrotjänster
- exponerar HTTP API
- körs som självständiga containers

### Frontend

- Vue‑appar byggs till statiska filer
- serveras via Nginx container

### Gemensamma egenskaper

- en image per repo
- immutabla versioner
- samma deploy‑modell oavsett teknik

---

## Deployment på server

Deployment sker genom att servern hämtar och startar nya images.

### Metod

```

docker compose pull
docker compose up -d

````

### Egenskaper

- ingen build sker på servern
- endast färdiga images används
- deterministiskt resultat

---

## Docker Compose

Servern använder `docker-compose` för att definiera tjänster.

### Exempel

```yaml
services:
  auth-service:
    image: ghcr.io/myorg/auth-service:latest

  time-service:
    image: ghcr.io/myorg/time-service:latest

  frontend:
    image: ghcr.io/myorg/frontend-ui:latest
````

***

## Routing

Routing hanteras av Traefik.

* baserat på domän eller hostnames
* varje container exponeras via HTTP
* Traefik dirigerar trafik till rätt tjänst

***

## Versionshantering

Alla deploymenter baseras på VERSION‑filen i respektive repo.

### Regler

VERSION måste uppdateras vid:

* ändringar i backend‑kod
* ändringar i frontend‑kod
* ändringar i OpenAPI‑kontrakt

### Syfte

* spårbarhet mellan kod och deploy
* tydlig versionshistorik
* stöd för rollback

### Exempel

```
VERSION = 1.3.0
```

→

```
ghcr.io/myorg/service:1.3.0
```

***

## Deployment‑strategi

Deployment sker normalt manuellt via SSH till servern.

### Flöde

1. kod mergas till `main`
2. CI bygger och publicerar image
3. ny version deployas genom att man på servern:

```
docker compose pull
docker compose up -d
```

***

### Fördelar

* enkel och robust
* full kontroll över release
* inga automatiska överraskningar
* lätt att felsöka

***

## Relation till CI

Deployment bygger på att CI har verifierat koden:

* Backend CI → kod och tester
* Frontend CI → bygg och kvalitet
* OpenAPI CI → kontrakt
* Security CI → beroenden

Endast godkänd kod i `main` deployas.

***

## Begränsningar

Deployment omfattar inte:

* databas‑migreringar
* backup‑strategier
* runtime‑monitoring
* autoskalning

Dessa ansvar hanteras separat.

***
## Databasmigreringar

Backend‑tjänster använder Phinx för att hantera databasmigreringar.

---

### Princip

Databasmigreringar är en integrerad del av applikationen och kopplas direkt till varje deploy.

Migreringar körs automatiskt vid container‑start för att säkerställa att databasschema alltid matchar applikationsversionen.

---

### Struktur

Varje backend‑repo innehåller migreringar, vanligtvis i:

/migrations

Exempel:

migrations/
  2024060101_create_tables.php
  2024060102_add_approved_at.php

---

### Körning

Migreringar körs automatiskt vid start av service‑containern.

Detta sker via containerns entrypoint:

- Phinx körs innan applikationen startar
- endast nya migreringar appliceras

---

### Flöde

1. kodändring inkluderar migration
2. CI bygger ny image
3. container deployas
4. container startar
5. Phinx kör migreringar
6. applikation startar

---

### Egenskaper

- migreringar körs alltid (ingen risk att glömmas)
- idempotent beteende via Phinx
- schema och kod hålls synkroniserade

---

### Viktiga principer

- migreringar ska vara bakåtkompatibla där möjligt
- undvik destruktiva ändringar i samma steg som deploy
- använd stegvis migrering (expand → migrate → cleanup)
- testa alltid migreringar lokalt innan deployment

---

### Begränsningar

Nuvarande lösning förutsätter:

- en instans per service
- ingen parallell uppstart av flera identiska containers

Vid skalning bör migrationshantering kompletteras med:

- låsning eller leader‑selection
- separat migrations‑container

---

### Sammanfattning

Databasmigreringar:

- hanteras via Phinx
- körs automatiskt vid container‑start
- kräver ingen manuell intervention

Detta säkerställer att:

- schema alltid är uppdaterat
- deployment är konsistent
- fel på grund av uteblivna migreringar undviks

---

## Sammanfattning

Deployment i plattformen bygger på:

* GitHub Actions för build
* GHCR som centralt image‑repository
* Docker Compose på servern
* pull‑baserad deployment

Det ger en lösning som är:

* enkel
* reproducerbar
* spårbar
* lätt att utöka
