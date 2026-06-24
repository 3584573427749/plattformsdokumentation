# Auth Service

## Syfte

Auth Service ansvarar för identitet, autentisering, auktorisering och användarhantering inom plattformen.

Tjänsten fungerar som den centrala källan för användare och behörigheter och används av samtliga övriga backend‑tjänster och klientapplikationer.

---

## Ansvarsområden

Auth Service ansvarar för:

* användarhantering
* autentisering
* auktorisering
* roller
* permissions
* JWT-hantering
* refresh tokens
* tokenrotation
* magic links
* OTP
* TOTP (tvåfaktorsautentisering)

Auth Service ansvarar inte för verksamhetsdata som grupper, tidrapporter eller resultat.

---

## Ägda data

Auth Service är system of record för identitetsrelaterad information.

### Users

Representerar användare i plattformen.

Exempel:

* id
* e-postadress
* förnamn
* efternamn
* aktiv/inaktiv status

### Roles

Definierar användarroller.

Exempel:

* administratör
* verksamhetsledare
* tränare
* funktionär

### Permissions

Representerar individuella rättigheter i systemet.

Permissions används för att bygga rollbaserad åtkomstkontroll (RBAC).

### Refresh Tokens

Används för sessionshantering och tokenförnyelse.

---

## User Management API

Auth Service exponerar ett API för hantering av användare.

### Funktioner

* lista användare
* hämta användare
* skapa användare
* uppdatera användare
* avaktivera användare

Detta API används primärt av Admin UI.

---

## Authentication API

Auth Service exponerar API:er för autentisering.

### Magic Links

Lösenordsfri inloggning via e‑post.

### OTP

Engångskoder för autentisering.

### TOTP

Autentisering baserad på autentiseringsappar.

### Refresh

Förnyelse av access tokens utan ny inloggning.

---

## Auktorisering

Auth Service implementerar rollbaserad åtkomstkontroll (RBAC).

Modellen består av:

```text
User
↓
Role
↓
Permission
```

Övriga tjänster använder information från Auth Service för att avgöra om en användare får utföra en viss operation.

---

## Tokenmodell

Auth Service använder:

* JWT access tokens
* httpOnly refresh tokens

Målet är att:

* minimera exponering av känsliga uppgifter
* möjliggöra kortlivade access tokens
* stödja tokenrotation

---

## Integrationer

### Gateway Service

Gateway Service verifierar och vidarebefordrar autentiserad trafik.

### Admin UI

Använder User Management API och administrativa funktioner.

### Övriga backend‑tjänster

Övriga tjänster förlitar sig på Auth Service för:

* identitet
* roller
* permissions

---

## Säkerhetsprinciper

Auth Service är en säkerhetskritisk komponent.

Följande principer gäller:

* OpenAPI är den enda källan till sanningen för API‑kontraktet
* samtliga endpoints valideras mot OpenAPI
* standardiserad felmodell används
* all autentisering sker via säkra tokens
* känslig data exponeras aldrig i API‑svar

---

## Databas

Auth Service äger sin egen databas.

Andra tjänster får aldrig skriva direkt till Auth Services databas.

Kommunikation ska alltid ske via publika API:er.

---

## Målsättning

Auth Service ska vara plattformens centrala identitets- och behörighetstjänst.

All autentisering, användarhantering och behörighetskontroll ska kunna spåras tillbaka till Auth Service och dess API-kontrakt.