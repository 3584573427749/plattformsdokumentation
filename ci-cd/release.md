# Release

Detta dokument beskriver hur tjänster och klientapplikationer releasas inom plattformen.

En release innebär att en verifierad artefakt byggs och publiceras. För backend‑tjänster innebär detta normalt en Docker‑image som publiceras till GitHub Container Registry (GHCR).

Release innebär inte automatiskt deployment till produktion.

---

## Översikt

Releaseflödet består av följande steg:

1. VERSION uppdateras
2. Kod mergeas till main
3. En git‑tagg skapas manuellt
4. CD startar
5. VERSION verifieras mot taggen
6. Docker‑image byggs
7. Docker‑image publiceras till GHCR

Efter detta kan deployment genomföras manuellt.

---

## VERSION som källa till sanningen

Varje tjänst innehåller en VERSION‑fil.

Exempel:

```text
1.2.3
```

VERSION‑filen är den enda källan till sanningen för tjänstens version.

Git‑taggar, Docker‑taggar och releaseartefakter måste överensstämma med innehållet i VERSION‑filen.

---

## Release under aktiv utveckling

Under aktiv utveckling skapas releaser manuellt.

Utvecklaren ansvarar för att:

1. Uppdatera VERSION
2. Skapa git‑tagg
3. Pusha git‑taggen

Exempel:

```bash
git tag v1.2.3
git push origin v1.2.3
```

Taggen fungerar som release‑trigger.

---

## Versionsverifiering

CD‑pipen verifierar att VERSION‑filen och git‑taggen överensstämmer.

Godkänt exempel:

```text
VERSION = 1.2.3
Tagg    = v1.2.3
```

Exempel som ska stoppas:

```text
VERSION = 1.2.3
Tagg    = v1.2.4
```

```text
VERSION = 1.2.4
Tagg    = v1.2.3
```

Ingen image publiceras om verifieringen misslyckas.

---

## GitHub Actions

Release initieras från respektive tjänst eller frontendprojekt.

Exempel:

```text
auth-service
└─ release-trigger

time-service
└─ release-trigger

group-service
└─ release-trigger
```

Gemensam release‑logik återanvänds från organisationens `.github`‑repository.

---

## Publicering till GHCR

Vid godkänd release byggs och publiceras en Docker‑image till GitHub Container Registry.

Exempel:

```text
ghcr.io/<organisation>/auth-service:1.2.3
ghcr.io/<organisation>/auth-service:latest
```

---

## Image‑taggar

Följande taggar publiceras:

```text
<service>:X.Y.Z
<service>:latest
```

Exempel:

```text
auth-service:1.2.3
auth-service:latest
```

### latest

latest representerar den senast publicerade releaseversionen och är den tagg som normalt används i produktion.

### Versionsspecifik tagg

X.Y.Z används för:

- felsökning
- rollback
- reproducerbara installationer

---

## Rollback

Eftersom varje release publiceras med en unik versionsetikett kan rollback ske genom att använda en tidigare image.

Exempel:

```text
auth-service:1.2.2
```

Ingen ny build krävs för rollback.

---

## Framtida automatisering

När plattformen når förvaltningsfas kan releaser automatiseras ytterligare.

Möjliga framtida förbättringar:

- automatisk skapande av git‑taggar
- automatisk generering av release notes
- mer avancerad versionhantering

VERSION‑filen förblir dock den enda källan till sanningen för tjänstens version.

---

## Begränsningar

Release omfattar inte:

- deployment
- databasmigreringar
- driftövervakning
- backuphantering

Dessa områden beskrivs i separat dokumentation.

---

## Sammanfattning

Releaseprocessen bygger på:

- VERSION som source of truth
- manuell git‑taggning
- automatisk verifiering i CD
- publicering till GHCR
- versionshanterade Docker‑images

Detta ger:

- full kontroll över releaser
- tydlig spårbarhet
- enkel rollback
- reproducerbara leveranser