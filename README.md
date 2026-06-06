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

Nameplates, discover, decorations i Giphy sa przygotowane konfiguracyjnie. Czesc tych funkcji zacznie dzialac dopiero po podmianie upstreamowych obrazow na wlasne obrazy Foxly z obsluga nowych endpointow i pol.
