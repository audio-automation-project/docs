# Configuration

Complete configuration reference for all services in the Audio Library System.

**First-time local startup:** use the ordered steps in [Local dev runbook](local-dev-runbook.md), then return here for variable detail.

## Backend Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | `8088` | HTTP port for the Spring Boot server |
| `DATABASE_URL` | `jdbc:postgresql://localhost:5432/audio_library` | JDBC connection string |
| `DATABASE_USER` | `audio` | Database username |
| `DATABASE_PASSWORD` | `audio` | Database password |
| `AUDIO_BASE_PATH` | `H:/Projects/.../data` | Root directory for audio file storage |
| `APP_CORE_ENABLED` | `true` | Feature flag — `true` = minimal mode, `false` = full mode |
| `FIREBASE_ENABLED` | `false` | Initialize Firebase Admin SDK |
| `GOOGLE_APPLICATION_CREDENTIALS` | — | Path to Firebase service account JSON (required if `firebase.enabled=true`) |
| `TELEGRAM_BOT_TOKEN` | placeholder | Telegram bot token (required if `core=false`) |
| `TORRENT_API_URL` | `http://localhost:8085` | qBittorrent Web API URL |
| `OPENAI_API_KEY` | — | OpenAI API key for DALL-E 3 and GPT-4o |

### Feature Flags

| Flag | Default | Effect |
|------|---------|--------|
| `app.core.enabled` | `true` | **`true`**: only Audio + Library active (minimal mode). **`false`**: all domains active (full mode) |
| `firebase.enabled` | `false` | `true` initializes Firebase Admin SDK |
| `firebase.firestore.sync` | `false` | `true` enables async Firestore sync after each SQL write |
| `app.migration.import-json` | `true` | One-shot JSON import on startup — set to `false` after first successful import |
| `app.migration.import-media-groups` | `false` | One-shot media group import — enable manually when needed |
| `spring.flyway.enabled` | `true` | Flyway database migration management |

### Spring Profiles

| Profile | Database | Use Case |
|---------|----------|----------|
| `default` | PostgreSQL | Production and standard development |
| `h2` | H2 in-memory | Quick local testing without PostgreSQL |

Activate a profile with: `mvn spring-boot:run -Dspring.profiles.active=h2`

### `application.properties` Example

```properties
server.port=${SERVER_PORT:8088}
spring.datasource.url=${DATABASE_URL:jdbc:postgresql://localhost:5432/audio_library}
spring.datasource.username=${DATABASE_USER:audio}
spring.datasource.password=${DATABASE_PASSWORD:audio}

app.core.enabled=${APP_CORE_ENABLED:true}
firebase.enabled=${FIREBASE_ENABLED:false}
firebase.firestore.sync=false

audio.base.path=${AUDIO_BASE_PATH:./data}
torrent.api.url=${TORRENT_API_URL:http://localhost:8085}
```

## Frontend Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `VITE_CORE_API_URL` | `http://localhost:8088` | Core backend URL for health checks |
| `VITE_SILENCE_API_URL` | `http://localhost:8090` | Python ML service URL for health checks |

Set in a `.env` file in the `audio-frontend/` root:

```
VITE_CORE_API_URL=http://localhost:8088
VITE_SILENCE_API_URL=http://localhost:8090
```

## Python ML Service Configuration

Configured via environment variables in `env.example`:

| Variable | Description |
|----------|-------------|
| Model paths | Paths to pretrained encoder, synthesizer, and vocoder models |
| CUDA settings | GPU device selection and memory settings |
| Audio settings | Sample rate, processing chunk sizes |

## External Service Dependencies

| Service | Required For | Default URL | Notes |
|---------|-------------|-------------|-------|
| FFmpeg | Audio + video processing | System PATH | Must be installed and accessible |
| PostgreSQL 15+ | Production database | `localhost:5432` | Not needed with `h2` profile |
| qBittorrent WebUI | Torrent management | `localhost:8085` | Only needed in full mode |
| Ollama | Local LLM inference | `localhost:8080` | Gemma model for search proxy |
| OpenAI API | DALL-E 3 + GPT-4o | `api.openai.com` | Requires API key |
| Playwright / Chromium | Boosty posting + scraping | Auto-downloaded | Only needed in full mode |
| Telegram Bot API | Bot commands | `api.telegram.org` | Requires bot token |
| Firebase/Firestore | Optional data projection | — | Requires service account JSON |
| NVIDIA CUDA | GPU acceleration for ML | — | Recommended for Python ML service |

## CORS Configuration

Defined in `config/CorsConfig.java`:

| Setting | Value |
|---------|-------|
| Allowed origins | `http://localhost:5173`, `http://localhost:4173` |
| Allowed methods | GET, POST, PUT, DELETE, PATCH, OPTIONS |
| Allowed headers | All |

## Related Documentation

- [Backend Architecture](../architecture/backend-architecture.md) — feature flag details
- [Local Development](local-development.md) — how to run with these configurations
