# Foxly Backend

Docker Compose stack dla Foxly.

## Deploy z GitHuba

Na serwerze:

```bash
git clone https://github.com/TWOJ_LOGIN/TWOJE_REPO.git foxly-backend
cd foxly-backend
cp .env.example .env
cp Revolt.toml.example Revolt.toml
cp livekit.yml.example livekit.yml
nano .env
nano Revolt.toml
nano livekit.yml
docker compose pull
docker compose up -d
```

Po update:

```bash
git pull
docker compose pull
docker compose up -d
```

## Pliki w repo

- `compose.yml` - uslugi Docker Compose
- `Caddyfile` - routing domen
- `.env.example` - szablon zmiennych
- `Revolt.toml.example` - szablon backend config
- `livekit.yml.example` - szablon LiveKit
- `ENDPOINTS.md` - mapa endpointow, payloadow i struktur MongoDB

Prawdziwe `.env`, `Revolt.toml` i `livekit.yml` nie powinny byc commitowane, bo zawieraja sekrety.

## Domeny

- `https://foxly.discordlabs.app` - web
- `https://api.foxly.discordlabs.app` - API, WebSocket, LiveKit, GIF proxy
- `https://files.foxly.discordlabs.app` - pliki
- `https://proxy.foxly.discordlabs.app` - proxy metadanych i obrazow

DNS tych domen powinien wskazywac na serwer.

## Porty

Otworz:

- `80/tcp`
- `443/tcp`
- `7881/tcp`
- `50000-50100/udp`

## Uwagi

Nameplates, discover, decorations, global activity i Giphy sa przygotowane konfiguracyjnie. Czesc tych funkcji zacznie dzialac dopiero po podmianie upstreamowych obrazow na wlasne obrazy Foxly z obsluga nowych endpointow i pol.

Wszystkie nowe funkcje Foxly maja byc doklejane do obecnego API, bez nowych domen:

- baza publiczna: `https://api.foxly.discordlabs.app/foxly/v1`
- baza wewnetrzna: `/foxly/v1`
- web: `VITE_FOXLY_API_BASE`

Proponowany kontrakt endpointow dla wlasnego obrazu API:

| Obszar | Endpointy |
| --- | --- |
| Chat | `GET /foxly/v1/channels/:id/messages/search`, `POST /foxly/v1/channels/:id/polls`, `POST /foxly/v1/channels/:id/scheduled-events`, `POST /foxly/v1/messages/:id/translate` |
| Watki i fora | `POST /foxly/v1/channels/:id/threads`, `GET /foxly/v1/channels/:id/threads`, `PATCH /foxly/v1/threads/:id`, `POST /foxly/v1/forums/:id/posts` |
| Voice | `GET /foxly/v1/voice/:channel_id/text`, `POST /foxly/v1/voice/:channel_id/clips`, `GET /foxly/v1/soundboard`, `POST /foxly/v1/soundboard/sounds` |
| Premium | `GET /foxly/v1/premium/plans`, `POST /foxly/v1/premium/checkout`, `POST /foxly/v1/premium/webhook`, `GET /foxly/v1/users/@me/entitlements` |
| Sklep | `GET /foxly/v1/shop/items`, `POST /foxly/v1/shop/purchase`, `POST /foxly/v1/gifts`, `POST /foxly/v1/credits/top-up` |
| Boosty | `GET /foxly/v1/servers/:id/boosts`, `POST /foxly/v1/servers/:id/boosts`, `DELETE /foxly/v1/servers/:id/boosts/:boost_id` |
| Aplikacje | `GET /foxly/v1/apps`, `POST /foxly/v1/apps`, `GET /foxly/v1/bots/directory`, `POST /foxly/v1/oauth/authorize` |
| Moderacja | `GET /foxly/v1/servers/:id/audit-log`, `POST /foxly/v1/servers/:id/automod/rules`, `POST /foxly/v1/reports`, `PATCH /foxly/v1/reports/:id` |
| Discover | `GET /foxly/v1/discover/featured`, `GET /foxly/v1/discover/categories`, `POST /foxly/v1/servers/:id/reviews` |
| Eksperymenty | `GET /foxly/v1/experiments`, `PATCH /foxly/v1/users/@me/experiments` |

Nowe zmienne sa w `.env` i `.env.example`, a limity oraz wlaczniki w `Revolt.toml` i `Revolt.toml.example`. Klucze platnosci, webhookow, translacji i analityki sa puste w szablonie i trzeba je uzupelnic dopiero na serwerze.

Mapa rozbudowanych obecnych endpointow, nowych tras Foxly i struktur MongoDB jest w `ENDPOINTS.md`.

Activity i status maja rozszerzac istniejacy model osoby/statusu, nie tworzyc osobnego modelu. Zapis zostaje w kolekcji `users` w MongoDB:

- update: `PATCH /users/@me`
- odczyt: `GET /users/:id/profile`
- status: `status`
- status since: `status_since`
- status update time: `status_updated_at`
- activity: `profile.activity`

Backend powinien przyjmowac payload:

```json
{
  "status": "dnd",
  "status_since": "ISO_DATE",
  "status_updated_at": "ISO_DATE",
  "activity": {
    "type": "playing",
    "name": "Visual Studio Code",
    "details": "Editing Foxly backend",
    "state": "Workspace: foxly",
    "platform": "desktop",
    "started_at": "ISO_DATE",
    "updated_at": "ISO_DATE"
  }
}
```

To powinno byc zapisane globalnie w istniejacym dokumencie usera, zeby kazdy klient widzial status i activity pod profilem uzytkownika.
