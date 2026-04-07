# OpenAPI-modell

Alla tjänster definierar sina API-kontrakt i `openapi.yaml`.
Frontend genererar typer från dessa.
CI säkerställer:
- Valid syntax
- Lint-fel
- Breaking changes
- Version bump
