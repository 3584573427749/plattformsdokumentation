# Branch-strategi

Plattformen använder trunk-based utveckling med kortlivade brancher.

## Brancher

### main

Innehåller alltid releasbar kod.

Direkta pushes till main är inte tillåtna.

All kod ska gå via Pull Request.

### feature/*

Används för ny funktionalitet.

Exempel:

feature/user-management
feature/openapi-validation

### fix/*

Används för buggrättningar.

Exempel:

fix/login-timeout
fix/user-delete

## Pull Requests

All kod ska granskas via Pull Request.

CI måste vara godkänd innan merge.

## Release

Merge till main skapar inte automatiskt en release.

Release sker genom att en git-tagg som motsvarar VERSION-filen skapas och pushas.