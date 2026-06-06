# Foxly Backend

Gotowy Docker stack dla Foxly.

## Pliki

- `.env` - domeny, tokeny, sekrety i flagi funkcji
- `compose.yml` - usługi Docker Compose
- `Caddyfile` - routing domen
- `Revolt.toml` - konfiguracja backendu
- `livekit.yml` - konfiguracja voice/video

## Domeny

Stack jest przygotowany pod:

- `https://foxly.discordlabs.app` - web
- `https://api.foxly.discordlabs.app` - API, WebSocket, LiveKit, GIF proxy
- `https://files.foxly.discordlabs.app` - pliki
- `https://proxy.foxly.discordlabs.app` - proxy metadanych i obrazów

DNS tych domen powinien wskazywać na serwer z Dockerem.

## Start

```bash
docker compose pull
docker compose up -d
```

## Porty

Otwórz na serwerze:

- `80/tcp`
- `443/tcp`
- `7881/tcp`
- `50000-50100/udp`

## Ważne

Nie usuwaj `.env`, `Revolt.toml` ani `livekit.yml` po starcie. `.env` zawiera sekrety do plików, push notifications i LiveKit.

Nameplates, discover, decorations i Giphy są przygotowane konfiguracyjnie. Część tych funkcji zadziała dopiero po użyciu własnych obrazów Foxly, które obsłużą nowe pola i endpoint `/nameplates`.
