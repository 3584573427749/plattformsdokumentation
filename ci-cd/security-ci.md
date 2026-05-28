# Security CI

Security CI ansvarar för att identifiera kända sårbarheter i plattformens externa beroenden.

CI‑flödet ger kontinuerlig övervakning av säkerhetsrisker i:

- backend‑tjänster (PHP)
- frontend‑applikationer (Node/npm)

Security CI är ett stödjande workflow och fungerar primärt som ett tidigt varningssystem.

---

## Aktivering

Security CI körs i repositories som representerar aktiva komponenter i plattformen:

- backend service‑repos (via `.ci/run-phpunit`)
- frontend‑repos (via `.ci/run-frontend-ci`)

Repositories utan dessa flaggor:

- betraktas som template‑ eller stöd‑repos
- kör inte security scan

---

## Funktionalitet

Security CI kontrollerar beroenden mot kända sårbarheter.

### Backend (PHP)

- kör `composer audit`
- analyserar beroenden i `composer.json`
- jämför med kända CVE‑databaser

---

### Frontend (Node)

- kör `npm audit`
- analyserar beroenden i `package.json`
- identifierar sårbara versioner

---

## Resultathantering

Security CI är i normalfallet **icke‑blockerande**:

- sårbarheter rapporteras i build‑loggen
- workflow fortsätter även vid upptäckta problem

Detta möjliggör:

- kontinuerlig utveckling
- samtidig synlighet av risker

---

## Syfte

Security CI säkerställer att:

- kända sårbarheter upptäcks tidigt
- beroenden hålls uppdaterade
- utvecklare informeras om risker innan deployment

---

## Rekommenderat arbetssätt

Vid rapporterade sårbarheter bör följande åtgärder övervägas:

- uppdatera beroenden till säkra versioner
- ersätta sårbara bibliotek
- utvärdera risknivå (t.ex. låg, moderat, hög)

Alla upptäckta sårbarheter kräver inte omedelbar åtgärd, men bör aktivt bedömas.

---

## Relation till övrig CI

Security CI kompletterar övriga CI‑flöden:

- Backend CI verifierar kod och tester
- Frontend CI verifierar bygg och funktion
- OpenAPI CI skyddar API‑kontraktet

Security CI fokuserar specifikt på externa beroenden.

---

## Begränsningar

Security CI omfattar inte:

- analys av egen kod (ingen statisk säkerhetsanalys)
- runtime‑säkerhet
- nätverks‑ eller infrastruktursäkerhet
- penetrationstestning

---

## Sammanfattning

Security CI är ett stödjande säkerhetslager som:

- identifierar sårbara beroenden
- ger tidig insyn i säkerhetsrisker
- möjliggör informerade beslut

Det är designat för att:

- ge hög synlighet med låg friktion
- inte blockera utveckling i tidiga faser
``
