# Utvecklingsfaser

Faserna beskriver plattformens övergripande utvecklingsmål och leveransmilstolpar.

Utvecklingsarbetet är iterativt och sker inte strikt linjärt. Det är vanligt att växla mellan flera repositories inom samma fas och att återvända till tidigare tjänster eller klienter för förbättringar, refactoring eller kompletterande funktionalitet.

En fas ska därför ses som ett verksamhetsmässigt delmål snarare än en exakt utvecklingsordning.

---

## Faser

| Fas | Innehåll | Mål |
|------|------|------|
| E1 | Auth Service, Gateway Service, Admin UI | Etablera plattformens grundläggande identitets- och administrationsfunktioner inklusive användarhantering (inklusive roller och rättigheter), autentisering och gemensamma UI-mönster |
| E2 | Time Service, Group Service, PWA Tränare | Hantering av grupper, gruppmedlemskap, ledarroller och tidrapportering |
| E3 | Results Service, Main UI | Resultathantering, personhistorik och publik webbplattform |
| E4 | Statistik, PWA Aktiva | Individuella vyer, statistik och funktionalitet för aktiva utövare |
| E5 | Competition Service, Entries UI | Tävlingar, grenar och anmälningshantering |

---

## Iterativ utveckling

Arbetet sker kontinuerligt mellan olika tjänster och klientapplikationer.

Exempelr

- Auth Service kan vidareutvecklas under E2–E5.
- Admin UI kan behöva kompletteras när nya backend-tjänster tillkommer.
- Gemensamma UI-komponenter och designmönster kan utvecklas parallellt med funktionalitet.
- Infrastruktur, CI/CD och dokumentation uppdateras löpande genom samtliga faser.

Faserna anger därför vad som ska vara etablerat för att nästa verksamhetsområde ska kunna utvecklas, inte att allt arbete inom en fas måste vara avslutat innan nästa fas påbörjas.