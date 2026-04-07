# Systemöversikt

Systemet är uppbyggt som en mikrotjänstplattform där varje tjänst är självständig, har egen databas och eget API-kontrakt.

## Huvudprinciper
- API-först (OpenAPI)
- Löst kopplade tjänster
- Ingen delad databas
- Gateway styr extern trafik
- Tokenbaserad auth
- Kontraktsstyrd utveckling
