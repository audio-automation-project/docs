# Gaps & TODOs — What Needs Work

Audit date: 2026-04-20. All items are grounded in specific files/classes found during source code inspection.

---

## Stubs to Implement

### YouTube Upload
- **File:** `poster/` — `YouTubeController.java`
- **Endpoint:** `POST /api/youtube/upload`
- **Current state:** Method body is commented-out placeholder code. Returns empty string.
- **What's missing:** OAuth 2.0 flow, YouTube Data API v3 video upload, metadata (title, description, tags).

### Ollama Integration
- **File:** `ai/` — Ollama controller/service
- **Endpoint:** `POST /api/ai/gemma`
- **Current state:** HTTP call is async fire-and-forget. Response is never read. Always returns `""`.
- **What's missing:** Await the Ollama HTTP response, return model output to caller.

---

## Incomplete Implementations

### Video Creation Pipeline
- **File:** `video/VideoCreation.java`
- **Endpoint:** `POST /api/video/create`
- **Current state:** Image-MP3 pair matching works. `VideoCreation.start()` is called but FFmpeg overlay composition (image + particle MP4 + audio) is not fully wired.
- **What's missing:** Complete the FFmpeg command that composites image frames, particle overlay, and audio into a valid MP4 output.

### OpenAI Image Cache Persistence
- **File:** `ai/AiTraceService.java`
- **Endpoint:** `POST /api/ai/openai/image/download`
- **Current state:** Cache is in-memory — lost on restart. `/download` endpoint is useless after a restart.
- **What's missing:** Persist generated image paths to PostgreSQL or to a file so downloads survive restarts.

---

## Security Issues

### Hardcoded OpenAI API Key
- **File:** `gateway/RestCallService.java`
- **Issue:** API key is a string literal in source code.
- **Fix:** Move to `application.properties` as `openai.api.key=${OPENAI_API_KEY}` and inject via `@Value`.

### Hardcoded Bearer Token (LLM Search Proxy)
- **File:** `search/` — HTTP header in search service
- **Issue:** Bearer token for the local LLM endpoint (`http://localhost:8080`) is a string literal.
- **Fix:** Same pattern — move to `application.properties` and inject via `@Value`.

---

## Code Quality

### Class Name Typos
| Typo | Correct Name | File |
|------|-------------|------|
| `ProperityConfig` | `PropertyConfig` | `file/ProperityConfig.java` — referenced everywhere |
| `ImageDescriptionTextExtractorSercvice` | `ImageDescriptionTextExtractorService` | `text/` package |

Both typos propagate through `@Autowired` injection sites. Rename would require updating all callers.

### Deprecated Telegram Endpoints
- **File:** `telegram/` — HTTP controller
- **Endpoints:** `GET /api/bot/audiobooks`, `POST /api/bot/audiobook`
- **Issue:** Marked `@Deprecated` but still active. Callers should use `/api/v1/library/audiobooks`.
- **Fix:** Remove or disable once all callers migrated.

---

## Missing Infrastructure

### No Flyway SQL Migration Files
- **Location:** `audio-library-automation-bot/src/main/resources/db/migration/`
- **Issue:** Directory is configured (`spring.flyway.locations=classpath:db/migration`) but contains no `.sql` files. Production schema depends on Hibernate `ddl-auto`, which is not safe for production (`update` can silently corrupt data).
- **Fix:** Add versioned Flyway migrations (`V1__create_audiobook.sql`, etc.) matching the current Hibernate entity definitions.

### Python ML Service — No REST API
- **Location:** `audio-silence-service/`
- **Issue:** Service runs as CLI/GUI (`demo_cli.py`, `demo_toolbox.py`) with no HTTP API. Port 8090 is referenced in the frontend health check and CLAUDE.md but no Flask/FastAPI server exists.
- **Fix:** Add a REST wrapper (e.g., FastAPI) with at minimum a `/actuator/health` endpoint and a voice cloning endpoint. Until then, health check from frontend will always fail.

### One-Shot Migration Flags Should Be Disabled After First Run
- **Flags:** `app.migration.import-json=true`, `app.migration.import-media-groups=false`
- **Issue:** `import-json` is `true` by default — every startup attempts a JSON import. After the initial migration this should be `false`.
- **Fix:** Set `app.migration.import-json=false` in `application.properties` after the first successful import.

### No Integration Test for Boosty Playwright Flow
- **Location:** `poster/` package
- **Issue:** Boosty automation uses Playwright with real browser sessions. There are no tests verifying the flow works — it can silently break when Boosty's HTML structure changes.
- **Fix:** At minimum, add a smoke test that verifies Playwright can launch and navigate to the login page.
