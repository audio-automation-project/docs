# Status Report — System Health Audit

Audit date: 2026-04-20. Based on direct source code inspection of `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/`.

---

## Fully Working

| Domain | Notes |
|--------|-------|
| **Audio** | All 4 endpoints (compress, trim, concatenate, duration) fully implemented with FFmpeg. Always enabled. |
| **Library (CRUD)** | Full JPA CRUD with transaction support, optional Firestore sync hook. Always enabled. |
| **Image** | Book cover generation via Java AWT fully implemented. |
| **Torrent** | Full qBittorrent Web API integration — login, poll, delete on completion. |
| **Search** | Google search proxy via local LLM endpoint working; URL extraction via regex. |
| **Parser (Baza-Knig)** | Playwright scraping with pagination, persistence to `parsed_book` table. |
| **Telegram** | Webhook bot functional; legacy HTTP endpoints work but are deprecated. |
| **Poster (Boosty)** | Playwright automation posts image+audio pairs to Boosty.ru. |
| **Text extraction** | Reads DB descriptions, sends to OpenAI GPT-4o, returns enhanced text. |
| **Firebase sync** | Async Firestore upsert after SQL commits — fully implemented but off by default. |
| **Gateway** | `RestCallService` inter-service orchestration fully implemented. |

---

## Partial — Works but Incomplete

### OpenAI Image (`/api/ai/openai/image`)
- Chat-to-image flow works (extracts text → DALL-E 3 → downloads PNG).
- `AiTraceService` cache is **in-memory only** — lost on restart.
- `/openai/image/download` returns cached results from the current session.

### Video Creation (`/api/video/create`)
- Controller and service exist; filename matching (PNG↔MP3) implemented.
- `VideoCreation.start()` FFmpeg overlay composition is **incomplete** — integration between service and FFmpeg command not finished.
- Returns response but may not produce valid MP4 output.

---

## Stubs — No Real Implementation

### Ollama (`/api/ai/gemma`)
- Sends prompt to `http://localhost:8080` with model `gemma:latest`.
- Does **not await the response** — fire-and-forget async call.
- Always returns empty string `""`.

### YouTube (`/api/youtube/upload`)
- Endpoint exists but the service body is **commented-out placeholder code only**.
- OAuth, upload, and metadata logic are missing.
- Returns empty string.

---

## Known Issues

### Security
| Issue | Location |
|-------|----------|
| OpenAI API key hardcoded | `gateway/RestCallService.java` |
| Bearer token hardcoded | `search/` service HTTP headers |
| Telegram bot token default is placeholder | `application.properties` (`telegram.bot.token`) |

### Code Quality
| Issue | Location |
|-------|----------|
| Class name typo: `ProperityConfig` | `file/ProperityConfig.java` — used everywhere |
| Class name typo: `ImageDescriptionTextExtractorSercvice` | `text/` package |
| 2 deprecated Telegram HTTP endpoints still live | `telegram/` controller |

### Infrastructure
| Issue | Notes |
|-------|-------|
| No Flyway SQL migration files in `db/migration/` | Schema exists via Hibernate `ddl-auto` only — risky for production |
| `app.migration.import-json` one-shot import | Runs on startup if `true`; should be disabled after first run |
| `app.migration.import-media-groups` default `false` | One-shot migration, must be enabled manually |

---

## External Dependency Requirements

| Dependency | Required For | Config Key | Default |
|------------|-------------|------------|---------|
| FFmpeg (system PATH) | All audio + video processing | — (PATH) | Must be installed |
| PostgreSQL | Production database | `DATABASE_URL` | `localhost:5432/audio_library` |
| Playwright / Chromium | Boosty posting, Baza-Knig scraping | — (auto-downloaded) | Required when `core=false` |
| qBittorrent WebUI | Torrent management | `TORRENT_API_URL` | `http://localhost:8085` |
| Ollama | AI/Gemma endpoint | — | `http://localhost:8080` |
| OpenAI API key | DALL-E 3 images, GPT-4o text | hardcoded in `RestCallService` | None (must set) |
| Google service account JSON | Firebase / Firestore sync | `GOOGLE_APPLICATION_CREDENTIALS` env var | Not required unless `firebase.enabled=true` |
| Telegram bot token | Telegram bot + reupload endpoint | `TELEGRAM_BOT_TOKEN` | Required when `core=false` |

---

## Feature Flag Summary

| Flag | Default | Effect when changed |
|------|---------|---------------------|
| `app.core.enabled` | `true` | `false` enables: Telegram, Boosty, AI, Torrent, Parser, Search, Video, Text, YouTube |
| `firebase.enabled` | `false` | `true` initializes Firebase Admin SDK |
| `firebase.firestore.sync` | `false` | `true` enables async Firestore sync after each SQL write |
| `app.migration.import-json` | `true` | Imports `audiobooks.json` to DB on startup — set to `false` after first run |
| `app.migration.import-media-groups` | `false` | One-shot media group import |
| `spring.flyway.enabled` | `true` | Manages DB schema migrations |
