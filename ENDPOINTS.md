# Foxly endpoint map

Ten plik opisuje kontrakt dla obecnych endpointow Revolt/Foxly po rozbudowie. Celem jest doklejanie wiekszej ilosci danych do istniejacych tras i kolekcji, bez tworzenia nowych domen.

## Zasada zgodnosci

- Istniejace endpointy zostaja pod `https://api.foxly.discordlabs.app`.
- Nowe pola sa doklejane do obecnych payloadow jako dodatkowe obiekty.
- Stary klient moze ignorowac nowe pola.
- Nowy klient powinien czytac pola Foxly, jezeli `FOXLY_EXISTING_ENDPOINTS_ENRICH_ENABLED=true`.
- Dane uzytkownika rozszerzamy glownie w `users`, dane serwerow w `servers`, dane wiadomosci w `messages`.
- Dane wrazliwe maja byc kompresowane bezstratnie przed szyfrowaniem i zapisywane jako envelope encryption.

## Obecne endpointy rozbudowane

| Endpoint | Za co odpowiada | Kolekcja MongoDB | Doklejane pola |
| --- | --- | --- | --- |
| `GET /users/@me` | Dane zalogowanego uzytkownika | `users` | `profile.activity`, `profile.nameplate`, `profile.decoration`, `premium`, `entitlements`, `devices`, `privacy`, `safety` |
| `PATCH /users/@me` | Aktualizacja profilu, statusu i activity | `users` | `status_since`, `status_updated_at`, `profile.activity`, `profile.pronouns`, `profile.location`, `privacy` |
| `GET /users/:id` | Publiczny profil uzytkownika | `users` | `badges`, `premium.public_badge`, `profile.activity`, `profile.nameplate`, `profile.decoration` |
| `GET /users/:id/profile` | Rozszerzony profil | `users` | `profile`, `status`, `premium`, `entitlements.public`, `mutual`, `safety.public_flags` |
| `GET /channels/:id/messages` | Lista wiadomosci | `messages` | `reactions_summary`, `attachments_meta`, `thread`, `poll`, `translation`, `moderation` |
| `GET /channels/:id/messages/:message_id` | Szczegoly jednej wiadomosci | `messages` | pelne `attachments_meta`, `thread`, `poll`, `analytics`, `moderation` |
| `POST /channels/:id/messages` | Wyslanie wiadomosci | `messages` | zapis `client_meta`, `attachments_meta`, `reply_context`, `safety_scan` |
| `GET /channels/:id` | Dane kanalu | `channels` | `stats`, `permissions_summary`, `thread_defaults`, `voice_text`, `last_activity_at` |
| `PATCH /channels/:id` | Konfiguracja kanalu | `channels` | `slowmode`, `thread_defaults`, `forum_tags`, `voice_text` |
| `GET /servers/:id` | Dane serwera | `servers` | `discover`, `boosts`, `premium`, `safety`, `features`, `stats`, `integrations` |
| `PATCH /servers/:id` | Konfiguracja serwera | `servers` | `discover`, `safety`, `features`, `premium_settings` |
| `GET /servers/:id/members` | Lista czlonkow | `server_members` | `roles_meta`, `activity_summary`, `safety_flags`, `premium_since` |
| `GET /servers/:id/members/:user_id` | Szczegoly czlonka | `server_members` | `joined_source`, `roles_meta`, `activity_summary`, `safety_flags` |
| `GET /auth/session/all` | Sesje uzytkownika | `sessions` | `device`, `client`, `ip_region`, `last_seen_at`, `risk_score` |
| `GET /attachments/:id` | Metadane pliku | `file_meta` | `width`, `height`, `duration_ms`, `blurhash`, `moderation`, `linked_resource` |

## Nowe endpointy Foxly pod obecnym API

| Endpoint | Za co odpowiada | Kolekcja MongoDB |
| --- | --- | --- |
| `GET /foxly/v1/channels/:id/messages/search` | Szukanie wiadomosci z filtrami | `messages`, indeks tekstowy |
| `POST /foxly/v1/channels/:id/polls` | Tworzenie ankiety w kanale | `messages`, pole `poll` |
| `POST /foxly/v1/channels/:id/threads` | Tworzenie watku | `threads`, `channels.thread_refs` |
| `GET /foxly/v1/channels/:id/threads` | Lista watkow kanalu | `threads` |
| `GET /foxly/v1/premium/plans` | Lista planow premium | `premium_plans` |
| `POST /foxly/v1/premium/checkout` | Start platnosci | `premium_checkouts` |
| `POST /foxly/v1/premium/webhook` | Webhook operatora platnosci | `premium_events`, `users.premium` |
| `GET /foxly/v1/users/@me/entitlements` | Uprawnienia i zakupione rzeczy | `entitlements` |
| `GET /foxly/v1/shop/items` | Sklep z kosmetykami i dodatkami | `shop_items` |
| `POST /foxly/v1/shop/purchase` | Zakup przedmiotu | `purchases`, `entitlements` |
| `GET /foxly/v1/servers/:id/audit-log` | Dziennik zdarzen serwera | `audit_log` |
| `POST /foxly/v1/reports` | Zgloszenia uzytkownikow/tresci | `reports` |

## Szyfrowanie i kompresja

Foxly powinien traktowac prywatne dane jako domyslnie szyfrowane. Nie ma jednego absolutnie "najlepszego" algorytmu dla kazdego przypadku, dlatego kontrakt wybiera mocny i praktyczny standard:

- AEAD dla danych: `AES-256-GCM`.
- Alternatywa dla bibliotek sodium/mobile: `XChaCha20-Poly1305`.
- Klucze per dokument/plik: envelope encryption.
- Rotacja master key: co 90 dni.
- Kompresja przed szyfrowaniem: `zstd`.
- Poziom hot data: `zstd level 6`.
- Poziom archiwum/backup: `zstd level 19`.
- Brak strat jakosci: nie transkodowac stratnie obrazow, audio ani video; oryginalny upload zostaje zachowany.

### Kolejnosc zapisu

```text
plain JSON/file
-> canonical serialize
-> zstd compress, jezeli rozmiar >= 1024 B i MIME nie jest juz skompresowany
-> encrypt AEAD z losowym nonce
-> store encrypted envelope
```

### Envelope dla zaszyfrowanego pola

```json
{
  "enc": {
    "v": 1,
    "alg": "AES-256-GCM",
    "kid": "foxly-local-v1",
    "nonce": "base64url",
    "aad": "collection:document_id:field_path",
    "ct": "base64url",
    "tag": "base64url",
    "compression": {
      "alg": "zstd",
      "level": 6,
      "original_size": 1234
    }
  }
}
```

### Pola szyfrowane w bazie

| Kolekcja | Pola szyfrowane | Powod |
| --- | --- | --- |
| `users` | `profile`, `privacy`, `devices`, `safety` | prywatny profil, ustawienia, urzadzenia, ryzyko konta |
| `messages` | `content`, `attachments_meta`, `safety_scan` | tresc wiadomosci i metadane zalacznikow |
| `sessions` | `device`, `ip_region`, `risk_score` | dane techniczne logowania |
| `reports` | caly dokument poza indeksami | zgloszenia moga zawierac prywatne tresci |
| `audit_log` | `metadata`, `reason`, `actor_ip_region` | dane administracyjne |
| `file_meta` | `owner_id`, `linked_resource`, `moderation` | powiazania plikow i wynik moderacji |

### Pola indeksowalne bez odszyfrowania

Szyfrowane pola nie nadaja sie do normalnego wyszukiwania tekstowego. Dla filtrowania nalezy utrzymywac bezpieczne pola pomocnicze:

```json
{
  "_id": "message_id",
  "channel": "channel_id",
  "author": "user_id",
  "content_enc": { "enc": {} },
  "search_shadow": {
    "content_hmac_terms": ["hmac_term_1", "hmac_term_2"],
    "locale": "pl",
    "created_day": "2026-06-06"
  }
}
```

`search_shadow` nie moze zawierac surowej tresci. Dla wyszukiwarki pelnotekstowej mozna uzyc osobnego indeksu tylko wtedy, gdy zostanie zaakceptowane przechowywanie znormalizowanych tokenow albo indeks bedzie szyfrowany aplikacyjnie.

### Kompresja plikow i mediow

| Typ danych | Strategia |
| --- | --- |
| JSON, logi, eksporty, backupy | `zstd`, kompresja bezstratna |
| PNG | zachowac oryginal, opcjonalnie wariant `webp lossless` |
| WebP | zachowac oryginal, nie pogarszac jakosci |
| JPEG/GIF/MP4/WebM/OGG/MP3 | nie kompresowac ponownie `zstd`, bo zwykle sa juz skompresowane |
| Miniatury | generowac osobne warianty, ale oryginal zostaje nienaruszony |
| Backups | `zstd level 19` + AES-256-GCM |

### Envelope dla pliku

```json
{
  "_id": "file_id",
  "bucket": "attachments",
  "storage": {
    "encrypted": true,
    "alg": "AES-256-GCM",
    "kid": "foxly-local-v1",
    "key_mode": "per-object",
    "nonce": "base64url",
    "compression": {
      "alg": "none",
      "reason": "mime_already_compressed"
    }
  },
  "integrity": {
    "sha256": "hex",
    "size_plain": 123456,
    "size_stored": 123900
  }
}
```

### Zasady implementacji

- Nigdy nie logowac plaintextu po odszyfrowaniu.
- Nie trzymac master key w repo.
- `FOXLY_SECURITY_MASTER_KEY` moze byc tylko lokalnym fallbackiem; produkcyjnie lepiej uzyc KMS/Vault.
- Nonce musi byc unikalny dla klucza.
- AEAD `aad` musi zawierac kolekcje, id dokumentu i sciezke pola, zeby wykryc podmiane ciphertextu.
- Kompresji nie wlaczac dla odpowiedzi zawierajacych sekrety zalezne od atakujacego, jezeli sa wspoldzielone z danymi kontrolowanymi przez atakujacego.
- Backupy musza byc kompresowane i szyfrowane przed wyslaniem poza serwer.

## Struktury w bazie

### `users`

```json
{
  "_id": "user_id",
  "username": "name",
  "status": "online",
  "status_since": "ISO_DATE",
  "status_updated_at": "ISO_DATE",
  "profile": {
    "enc": {
      "v": 1,
      "alg": "AES-256-GCM",
      "kid": "foxly-local-v1",
      "nonce": "base64url",
      "ct": "base64url",
      "tag": "base64url",
      "compression": { "alg": "zstd", "level": 6 }
    },
    "decrypted_shape": {
      "content": "bio",
      "pronouns": "he/him",
      "location": "PL",
      "activity": {
        "type": "playing",
        "name": "Foxly",
        "details": "In voice channel",
        "state": "Server name",
        "platform": "desktop",
        "started_at": "ISO_DATE",
        "updated_at": "ISO_DATE"
      },
      "nameplate": {
        "asset_id": "file_id",
        "preset_id": "premium_gold",
        "animated": true
      },
      "decoration": {
        "asset_id": "file_id",
        "expires_at": null
      }
    }
  },
  "badges": ["early_user", "premium"],
  "premium": {
    "active": true,
    "plan": "premium_monthly",
    "renews_at": "ISO_DATE",
    "public_badge": true
  },
  "entitlements": ["nameplate_gold", "upload_500mb"],
  "devices": {
    "enc": {
      "v": 1,
      "alg": "AES-256-GCM",
      "kid": "foxly-local-v1",
      "nonce": "base64url",
      "ct": "base64url",
      "tag": "base64url",
      "compression": { "alg": "zstd", "level": 6 }
    },
    "decrypted_shape": [
      {
        "id": "device_id",
        "type": "desktop",
        "last_seen_at": "ISO_DATE"
      }
    ]
  },
  "privacy": {
    "enc": {
      "v": 1,
      "alg": "AES-256-GCM",
      "kid": "foxly-local-v1",
      "nonce": "base64url",
      "ct": "base64url",
      "tag": "base64url"
    },
    "decrypted_shape": {
      "show_activity": true,
      "allow_dm_from_servers": true
    }
  },
  "safety": {
    "enc": {
      "v": 1,
      "alg": "AES-256-GCM",
      "kid": "foxly-local-v1",
      "nonce": "base64url",
      "ct": "base64url",
      "tag": "base64url"
    },
    "decrypted_shape": {
      "risk_score": 0,
      "flags": []
    }
  }
}
```

Publiczny payload API po odszyfrowaniu przez backend:

```json
{
  "_id": "user_id",
  "username": "name",
  "status": "online",
  "profile": {
    "content": "bio",
    "activity": {
      "type": "playing",
      "name": "Foxly",
      "details": "In voice channel",
      "state": "Server name",
      "platform": "desktop",
      "started_at": "ISO_DATE",
      "updated_at": "ISO_DATE"
    },
    "nameplate": {
      "asset_id": "file_id",
      "preset_id": "premium_gold",
      "animated": true
    },
    "decoration": {
      "asset_id": "file_id",
      "expires_at": null
    }
  },
  "badges": ["early_user", "premium"],
  "premium": {
    "active": true,
    "plan": "premium_monthly",
    "renews_at": "ISO_DATE",
    "public_badge": true
  },
  "entitlements": ["nameplate_gold", "upload_500mb"],
  "devices": [
    {
      "id": "device_id",
      "type": "desktop",
      "last_seen_at": "ISO_DATE"
    }
  ],
  "privacy": {
    "show_activity": true,
    "allow_dm_from_servers": true
  }
}
```

### `messages`

```json
{
  "_id": "message_id",
  "channel": "channel_id",
  "author": "user_id",
  "content_enc": {
    "enc": {
      "v": 1,
      "alg": "AES-256-GCM",
      "kid": "foxly-local-v1",
      "nonce": "base64url",
      "ct": "base64url",
      "tag": "base64url",
      "compression": { "alg": "zstd", "level": 6 }
    }
  },
  "content_decrypted_shape": "hello",
  "attachments": ["file_id"],
  "attachments_meta_enc": {
    "enc": {
      "v": 1,
      "alg": "AES-256-GCM",
      "kid": "foxly-local-v1",
      "nonce": "base64url",
      "ct": "base64url",
      "tag": "base64url",
      "compression": { "alg": "zstd", "level": 6 }
    },
    "decrypted_shape": [
      {
        "file_id": "file_id",
        "mime": "image/png",
        "width": 1200,
        "height": 800,
        "blurhash": "hash",
        "moderation": {
          "status": "clean"
        }
      }
    ]
  },
  "reactions_summary": {
    "total": 12,
    "top": [{ "emoji": "fire", "count": 8 }]
  },
  "thread": {
    "id": "thread_id",
    "reply_count": 4,
    "last_message_at": "ISO_DATE"
  },
  "poll": {
    "question": "Choose",
    "options": [{ "id": "a", "text": "A", "votes": 1 }],
    "expires_at": "ISO_DATE"
  },
  "translation": {
    "source_locale": "pl",
    "available": ["en", "de"]
  },
  "moderation": {
    "status": "clean",
    "matched_rules": []
  }
}
```

### `servers`

```json
{
  "_id": "server_id",
  "name": "Foxly",
  "owner": "user_id",
  "features": ["discover", "threads", "premium"],
  "discover": {
    "enabled": true,
    "category": "community",
    "tags": ["chat", "voice"],
    "score": 98,
    "review_count": 42
  },
  "boosts": {
    "level": 2,
    "count": 14,
    "perks": ["hd_voice", "vanity_assets"]
  },
  "premium": {
    "vanity_url": "foxly",
    "banner_enabled": true
  },
  "safety": {
    "automod_enabled": true,
    "default_message_scan": true
  },
  "stats": {
    "members": 1200,
    "online": 300,
    "messages_24h": 9000
  },
  "integrations": {
    "webhooks": 4,
    "bots": 12
  }
}
```

### `channels`

```json
{
  "_id": "channel_id",
  "server": "server_id",
  "channel_type": "Text",
  "name": "general",
  "stats": {
    "messages_24h": 123,
    "active_threads": 7
  },
  "permissions_summary": {
    "can_send": true,
    "can_attach": true,
    "can_create_threads": true
  },
  "thread_defaults": {
    "auto_archive_minutes": 1440,
    "slowmode_seconds": 0
  },
  "voice_text": {
    "enabled": true,
    "linked_voice_channel_id": "voice_id"
  },
  "last_activity_at": "ISO_DATE"
}
```

### `server_members`

```json
{
  "_id": "server_id:user_id",
  "server": "server_id",
  "user": "user_id",
  "roles": ["role_id"],
  "roles_meta": [
    {
      "id": "role_id",
      "name": "Admin",
      "color": "#ffcc00"
    }
  ],
  "activity_summary": {
    "last_message_at": "ISO_DATE",
    "last_voice_at": "ISO_DATE"
  },
  "joined_source": "invite_code",
  "premium_since": "ISO_DATE",
  "safety_flags": []
}
```

### `sessions`

```json
{
  "_id": "session_id",
  "user": "user_id",
  "device": {
    "type": "desktop",
    "os": "Windows",
    "browser": "Chrome"
  },
  "client": {
    "name": "Foxly Web",
    "version": "1.0.0"
  },
  "ip_region": "PL",
  "last_seen_at": "ISO_DATE",
  "risk_score": 0
}
```

### `file_meta`

```json
{
  "_id": "file_id",
  "bucket": "attachments",
  "owner_id": "user_id",
  "mime": "image/png",
  "size": 123456,
  "width": 1200,
  "height": 800,
  "duration_ms": null,
  "blurhash": "hash",
  "moderation": {
    "status": "clean",
    "checked_at": "ISO_DATE"
  },
  "linked_resource": {
    "type": "message",
    "id": "message_id"
  }
}
```

## Minimalne indeksy MongoDB

```js
db.users.createIndex({ "status": 1, "profile.activity.updated_at": -1 })
db.messages.createIndex({ "channel": 1, "_id": -1 })
db.messages.createIndex({ "content": "text" })
db.messages.createIndex({ "thread.id": 1 })
db.servers.createIndex({ "discover.enabled": 1, "discover.score": -1 })
db.server_members.createIndex({ "server": 1, "user": 1 }, { unique: true })
db.sessions.createIndex({ "user": 1, "last_seen_at": -1 })
db.file_meta.createIndex({ "owner_id": 1, "linked_resource.type": 1, "linked_resource.id": 1 })
```
