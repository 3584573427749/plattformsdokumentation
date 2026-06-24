# Deployment

Deployment ansvarar för att installera och uppdatera redan publicerade versioner i driftmiljön.

Deployment är ett manuellt steg och initieras aldrig automatiskt av CI/CD.

Release och deployment är två separata processer:

Release
→ bygger och publicerar artefakter

Deployment
→ installerar publicerade artefakter i driftmiljön

---

## Grundprinciper

Deployment ska:

- använda färdigbyggda Docker-images från GHCR
- aldrig bygga kod på servern
- vara reproducerbart
- vara enkelt att rollbacka

Produktion använder normalt image-taggen:

```text
latest
````

Exempel:

```yaml
services:
  auth-service:
    image: ghcr.io/<organisation>/auth-service:latest
```

***

## Normal deployment

På produktionsservern körs:

```bash
docker compose pull
docker compose up -d
```

Detta hämtar senaste publicerade images och startar om berörda tjänster.

***

## Databasmigreringar

Backend‑tjänster använder Phinx för att hantera databasmigreringar.

Migreringar körs automatiskt vid varje containerstart innan applikationen startas. Detta säkerställer att databasschemat alltid är synkroniserat med den version av tjänsten som körs.

Principer:

* varje tjänst migrerar endast sin egen databas
* inga cross-service-migreringar får förekomma
* migreringar hanteras med Phinx
* endast migreringar som ännu inte har körts appliceras
* migreringar ska vara idempotenta och säkra att köra flera gånger

Förenklat startflöde:

```text
Container startar
↓
Phinx kör migreringar
↓
Applikationen startar
````

Det innebär att ingen manuell migreringskörning normalt krävs vid deployment.

För att en migrering ska kunna genomföras säkert bör den vara bakåtkompatibel och kunna köras utan att påverka övriga tjänster i plattformen.


***

## Rollback

Vid problem kan en tidigare image-version användas.

Exempel:

```yaml
services:
  auth-service:
    image: ghcr.io/<organisation>/auth-service:1.2.3
```

Därefter:

```bash
docker compose up -d
```

Ingen ny build krävs eftersom samtliga releaser publiceras med en unik versionsetikett.

***

## Ansvarsfördelning

### CI

Ansvarar för:

* tester
* kvalitetssäkring
* OpenAPI-kontroller

### Release

Ansvarar för:

* byggnation av Docker-images
* publicering till GHCR

### Deployment

Ansvarar för:

* installation av publicerade images
* uppdatering av driftmiljö
* körning av migreringar

