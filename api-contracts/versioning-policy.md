# Versioneringspolicy (SemVer)

Samtliga tjänster och klientapplikationer använder Semantic Versioning (SemVer):

MAJOR.MINOR.PATCH

Exempel:

1.0.0
1.2.0
1.2.3

## VERSION-filen

Varje tjänst innehåller en VERSION-fil som är den enda källan till sanningen för tjänstens version.

Exempel:

1.2.3

## PATCH

PATCH-versionen ska ökas för:

- Buggrättningar
- Interna kodförbättringar
- Prestandaförbättringar
- Ändringar som inte påverkar API-kontraktet

Exempel:

1.2.3 → 1.2.4

## MINOR

MINOR-versionen ska ökas för:

- Nya endpoints
- Nya fält i API-responser
- Nya funktioner som är bakåtkompatibla
- Utökad affärslogik som inte bryter befintliga klienter

Exempel:

1.2.3 → 1.3.0

## MAJOR

MAJOR-versionen ska ökas för:

- Breaking changes i API-kontraktet
- Borttagna endpoints
- Borttagna fält
- Inkompatibla förändringar av DTO:er
- Förändringar som kräver ändringar i klientapplikationer

Exempel:

1.2.3 → 2.0.0

## OpenAPI-kontroll

OpenAPI-diff används för att identifiera kontraktsförändringar.

CI/CD ska stoppa förändringar där:

- Breaking changes saknar MAJOR-bump
- API-förändringar saknar versionsuppdatering

## Releaser

Under aktiv utveckling skapas releaser manuellt genom att skapa och pusha en git-tagg som motsvarar innehållet i VERSION-filen.

Exempel:

VERSION:
1.2.3

Tagg:
v1.2.3

CD-pipelinen ska verifiera att VERSION-filen och git-taggen överensstämmer innan en release publiceras.