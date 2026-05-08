# REST API Reference

Complete endpoint inventory for the Audio Library System backend (`http://localhost:8088`).

The backend operates in two modes controlled by the `app.core.enabled` feature flag:
- **`true`** (default) — only Audio and Library endpoints are active
- **`false`** — all endpoints are active

## Audio Processing — `/api/audio`

*Always active.*

### POST `/api/audio/compress`

Compresses all MP3 files in a directory to 128kbps.

| Aspect | Detail |
|--------|--------|
| **Request body** | `{ "path": "<directory>" }` |
| **What it does** | Iterates all MP3 files in the specified directory, compresses each to 128k bitrate via FFmpeg |
| **Output** | Compressed files written to `audioC/` subdirectory |
| **Requires** | FFmpeg on system PATH |

### POST `/api/audio/concatenate`

Concatenates all MP3 files in a directory into a single file.

| Aspect | Detail |
|--------|--------|
| **Request body** | `{ "path": "<directory>" }` |
| **What it does** | Reads all MP3 files in the directory, concatenates them in order, generates a metadata JSON with chapter durations |
| **Output** | Single concatenated MP3 + `description.json` metadata |
| **Requires** | FFmpeg on system PATH |

### POST `/api/audio/duration`

Returns the duration of an audio file.

| Aspect | Detail |
|--------|--------|
| **Request body** | `{ "path": "<file.mp3>" }` |
| **Response** | Duration string in format `"Xч Yм Zс"` |
| **Requires** | FFmpeg on system PATH |

### POST `/api/audio/trim`

Trims an audio file from a specified start time.

| Aspect | Detail |
|--------|--------|
| **Request body** | `{ "path": "<file.mp3>", "startSeconds": N }` |
| **What it does** | Trims the audio file starting from `startSeconds` |
| **Output** | Trimmed MP3 file |
| **Requires** | FFmpeg on system PATH |

---

## Audiobook Library — `/api/v1/library/audiobooks`

*Always active.*

### GET `/api/v1/library/audiobooks`

Returns all audiobooks in the catalog.

| Aspect | Detail |
|--------|--------|
| **Response** | JSON array of `AudiobookDto` objects |

### GET `/api/v1/library/audiobooks/{id}`

Returns a single audiobook by ID.

| Aspect | Detail |
|--------|--------|
| **Path param** | `id` — audiobook identifier |
| **Response** | `AudiobookDto` JSON or `404 Not Found` |

### POST `/api/v1/library/audiobooks`

Creates a new audiobook record.

| Aspect | Detail |
|--------|--------|
| **Request body** | `AudiobookDto` JSON |
| **Response** | `201 Created` + created `AudiobookDto` |
| **Side effect** | If `firebase.firestore.sync=true`, asynchronously syncs to Firestore |

### PUT `/api/v1/library/audiobooks/{id}`

Updates an existing audiobook.

| Aspect | Detail |
|--------|--------|
| **Path param** | `id` — audiobook identifier |
| **Request body** | `AudiobookDto` JSON |
| **Response** | `200 OK` + updated `AudiobookDto` or `404 Not Found` |
| **Side effect** | If `firebase.firestore.sync=true`, asynchronously syncs to Firestore |

### DELETE `/api/v1/library/audiobooks/{id}`

Deletes an audiobook.

| Aspect | Detail |
|--------|--------|
| **Path param** | `id` — audiobook identifier |
| **Response** | `204 No Content` or `404 Not Found` |
| **Side effect** | If `firebase.firestore.sync=true`, asynchronously deletes from Firestore |

### AudiobookDto (PostgreSQL `library_audiobooks`)

Persisted by **`LibraryAudiobookService`** / **`AudiobookV1Controller`**. DDL source of truth: Flyway **`V202605052210__library_audiobooks.sql`**.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | **`catalog_public_id`** — public catalog id (not the bigint surrogate key) |
| `fileId` | string | Telegram file identifier (when sourced from Telegram) |
| `title` | string | Title |
| `author` | string | Author |
| `narrator` | string | Narrator / reader |
| `originalTitle` | string | Original title |
| `part` | string | Part label |
| `sourceLink` | string | Source URL |
| `duration` | number (long) | Duration in seconds |
| `uploadDate` | string | Upload date (opaque string) |
| `description` | string | Description |
| `coverImageIds` | string[] | Cover image document ids (historical / mirror-related list) |
| `distributedTo` | object | **`youtubeVideoId`**, **`telegramFileId`** only (see below) |
| `processingStatus` | string | e.g. `IDLE`, `PROCESSING`, `DISTRIBUTED`, `FAILED` |

**`distributedTo`** (embedded **`DistributedToDto`**):

| Field | Type | Maps to column |
|-------|------|----------------|
| `youtubeVideoId` | string | `dist_youtube_video_id` |
| `telegramFileId` | string | `dist_telegram_file_id` |

---

## Image Generation — `/api/book-cover`

*Requires `app.core.enabled=false`.*

### POST `/api/book-cover/generate`

Generates book cover images with text overlay.

| Aspect | Detail |
|--------|--------|
| **Request body** | `{ "imagePath": "<file>", "title": "...", "count": N }` |
| **What it does** | Takes a base image, renders "АУДИОКНИГА" header, title text + number, and "~~~~~~~~~~2024~~~~~~~~~~" footer in Comic Sans MS Bold |
| **Output** | N PNG files written to `image/` (named `1.png`, `2.png`, …) |

---

## Torrent Management — `/api/torrent`

*Requires `app.core.enabled=false`.*

### POST `/api/torrent/upload`

Uploads torrent files to qBittorrent.

| Aspect | Detail |
|--------|--------|
| **Request body** | None (reads from configured `torrentsPath`) |
| **Response** | Response JSON |
| **Requires** | qBittorrent running at `TORRENT_API_URL` (default `http://localhost:8085`) |

### POST `/api/torrent/monitor`

Polls torrent download status until completion.

| Aspect | Detail |
|--------|--------|
| **Response** | `MonitorTorrents` JSON — list of torrents + completion flag |
| **Behavior** | Polls until all torrents reach `stalledUp` or `uploading` state, then deletes them from qBittorrent |

### POST `/api/torrent/monitor/enable`

Enables the torrent monitoring scheduler.

### POST `/api/torrent/monitor/disable`

Disables the torrent monitoring scheduler.

---

## Catalog Search — `/api/search`

*Requires `app.core.enabled=false`.*

### GET `/api/search`

Searches for audiobook sources via LLM proxy.

| Aspect | Detail |
|--------|--------|
| **Query param** | `query=<text>` |
| **Response** | `SearchInformation` — list of extracted URLs |
| **How it works** | Proxies through Ollama-compatible endpoint at `http://localhost:8080/api/chat/completions` using model `live_search.google`. Extracts URLs from response via regex. |

---

## Catalog Parser — `/api/parser`

*Requires `app.core.enabled=false`.*

### POST `/api/parser/parse`

Scrapes audiobook metadata from baza-knig.ru.

| Aspect | Detail |
|--------|--------|
| **Request body** | `SearchInformation` (URLs + query) |
| **Response** | `BookInkParser` (parsed books + metadata) |
| **How it works** | Launches Playwright/Chromium, navigates to each URL, scrapes book metadata |
| **Side effect** | Stores results in `parsed_book` table |

### POST `/api/parser/parse/part`

Paginated version of the parser.

| Aspect | Detail |
|--------|--------|
| **Request body** | `{ "url": "...", "query": "..." }` |
| **Response** | `BookInkParser` (paginated results) |

---

## Telegram Bot — `/api/bot`

*Requires `app.core.enabled=false`.*

### POST `/api/bot/reupload/{fileId}`

Re-uploads a file to a Telegram chat.

| Aspect | Detail |
|--------|--------|
| **Path param** | `fileId` — Telegram file identifier |
| **Query param** | `chatId` — target chat ID |
| **Response** | `204 No Content` or `503 Service Unavailable` |

### Webhook Bot Commands

The `AudioLibraryBot` handles these Telegram commands:

| Command | What It Does |
|---------|-------------|
| `/list` | List all audiobooks |
| `/get` | Get audiobook details |
| `/upload` | Upload audiobook to Telegram |
| `/reupload` | Re-upload audiobook to a different chat |

---

## Boosty Publishing — `/api/boosty`

*Requires `app.core.enabled=false`.*

### POST `/api/boosty/post`

Publishes content to Boosty.ru via browser automation.

| Aspect | Detail |
|--------|--------|
| **Request body** | `{ "imageDir": "...", "audioDir": "...", "cicleName": "..." }` |
| **Response** | `"success"` string |
| **How it works** | Uses Playwright to automate a Chromium browser session — logs in, creates a post, matches image/audio pairs by filename, uploads them, adds tags |
| **Tags** | аудиокнига, дар, магия, фантастика, попаданец, фэнтези, слушать фэнтези, аудиокниги онлайн |
| **Requires** | Playwright/Chromium, valid Boosty session |

---

## YouTube — `/api/youtube`

*Requires `app.core.enabled=false`.*

### POST `/api/youtube/upload`

YouTube upload endpoint — not yet implemented. Returns empty string.

---

## Text Enhancement — `/api/text`

*Requires `app.core.enabled=false`.*

### POST `/api/text/description`

Enhances audiobook descriptions using AI.

| Aspect | Detail |
|--------|--------|
| **Request body** | None (reads from configured `torrentsPath`) |
| **Response** | `ExtractedDescription` JSON — enhanced description text |
| **How it works** | Reads `ParsedBookEntity` records from DB, concatenates descriptions, sends to OpenAI GPT-4o (2500 token limit, 0.3 temperature) for enhancement |

---

## AI / OpenAI — `/api/ai/openai`

*Requires `app.core.enabled=false`.*

### POST `/api/ai/openai/image`

Generates images using DALL-E 3.

| Aspect | Detail |
|--------|--------|
| **Request body** | `OpenAiResponse` (chat completion JSON) |
| **What it does** | Extracts text from the chat completion, sends it to DALL-E 3 (1792×1024, vivid style), downloads the result |
| **Output** | PNG image + download URL |

### POST `/api/ai/openai/image/download`

Downloads previously generated AI images from the current session cache.

---

## AI / Ollama — `/api/ai`

*Requires `app.core.enabled=false`.*

### POST `/api/ai/gemma`

Sends a prompt to local Ollama.

| Aspect | Detail |
|--------|--------|
| **Request body** | Text string |
| **Response** | Empty string (fire-and-forget) |
| **How it works** | Sends prompt to `http://localhost:8080` with model `gemma:latest` |

---

## Video Composition — `/api/video`

*Requires `app.core.enabled=false`.*

### POST `/api/video/create`

Composites images and audio into video files.

| Aspect | Detail |
|--------|--------|
| **Request body** | PNG image directory, MP4 particle overlay, MP3 audio directory |
| **What it does** | Matches PNG images to MP3 audio by filename, calls `VideoCreation.start()` per pair |
| **Output** | MP4 video files in `video/` directory |

---

## Health Check

*Always active (Spring Boot Actuator).*

### GET `/actuator/health`

Returns service health status.

| Aspect | Detail |
|--------|--------|
| **Response** | `{"status": "UP"}` when healthy |
| **Used by** | Frontend dashboard for service monitoring |

## Related Documentation

- [Backend Architecture](../architecture/backend-architecture.md) — controller and service layer details
- [Audiobook Pipeline](../pipeline/audiobook-pipeline.md) — how endpoints connect in the production workflow
- [DTO Catalog](../data/dto-catalog.md) — request/response data structures
