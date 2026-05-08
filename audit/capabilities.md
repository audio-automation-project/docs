# Capabilities — REST Endpoint Inventory

All endpoints are on the core backend (`http://localhost:8088`).

Feature flag: `app.core.enabled` in `application.properties`.
- `true` (default) — only **Always-on** domains active
- `false` — all domains active

---

## Always-on (app.core.enabled=true)

### Audio — `/api/audio`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/audio/compress` | **REAL** | `{ "path": "<dir>" }` | Compressed MP3 files (128k) written to `audioC/` |
| POST | `/api/audio/concatenate` | **REAL** | `{ "path": "<dir>" }` | Single concatenated MP3 + metadata JSON |
| POST | `/api/audio/duration` | **REAL** | `{ "path": "<file.mp3>" }` | Duration string `"Xч Yм Zс"` |
| POST | `/api/audio/trim` | **REAL** | `{ "path": "<file.mp3>", "startSeconds": N }` | Trimmed MP3 file |

FFmpeg is required on the system PATH for all audio operations.

### Library — `/api/v1/library/audiobooks`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| GET | `/api/v1/library/audiobooks` | **REAL** | — | `[AudiobookDto]` list |
| GET | `/api/v1/library/audiobooks/{id}` | **REAL** | `id` path param | `AudiobookDto` or 404 |
| POST | `/api/v1/library/audiobooks` | **REAL** | `AudiobookDto` JSON body | 201 + created `AudiobookDto` |
| PUT | `/api/v1/library/audiobooks/{id}` | **REAL** | `id` + `AudiobookDto` body | 200 + updated `AudiobookDto` or 404 |
| DELETE | `/api/v1/library/audiobooks/{id}` | **REAL** | `id` path param | 204 or 404 |

`AudiobookDto` fields: `id`, `fileId`, `title`, `author`, `narrator`, `originalTitle`, `part`, `sourceLink`, `duration`, `audioFilePath`, `coverImagePath`, `uploadDate`, `description`.

Optional: if `firebase.firestore.sync=true`, each write asynchronously syncs to Firestore.

---

## Requires app.core.enabled=false

### Image — `/api/book-cover`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/book-cover/generate` | **REAL** | `{ "imagePath": "<file>", "title": "...", "count": N }` | N PNG files written to `image/` (named `1.png`, `2.png`, …) |

Uses Java AWT. Renders "АУДИОКНИГА" header, title + number, and "~~~~~~~~~~2024~~~~~~~~~~" footer in Comic Sans MS Bold.

### Torrent — `/api/torrent`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/torrent/monitor` | **REAL** | — | `MonitorTorrents` JSON (list + completion flag) |
| POST | `/api/torrent/monitor/enable` | **REAL** | — | Status string |
| POST | `/api/torrent/monitor/disable` | **REAL** | — | Status string |
| POST | `/api/torrent/upload` | **REAL** | — (reads `torrentsPath` from config) | Response JSON |

Requires qBittorrent running at `torrent.api.url` (default `http://localhost:8085`). Monitor polls until all torrents reach `stalledUp` or `uploading` state, then deletes them.

### Search — `/api/search`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| GET | `/api/search` | **REAL** | `?query=<text>` | `SearchInformation` (list of URLs) |

Proxies via Ollama-compatible endpoint at `http://localhost:8080/api/chat/completions` using model `live_search.google`. Extracts URLs from response via regex.

### Parser — `/api/parser`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/parser/parse` | **REAL** | `SearchInformation` (URLs + query) | `BookInkParser` (books + metadata) |
| POST | `/api/parser/parse/part` | **REAL** | `{ "url": "...", "query": "..." }` | `BookInkParser` (paginated) |

Scrapes `baza-knig.ru` using Playwright (Chromium). Stores results in `parsed_book` table. Requires Chromium binary.

### Telegram — `/api/bot`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| GET | `/api/bot/audiobooks` | **REAL** (deprecated) | — | `[Audiobook]` JSON |
| POST | `/api/bot/audiobook` | **REAL** (deprecated) | `Audiobook` JSON | 200 no body |
| POST | `/api/bot/reupload/{fileId}` | **REAL** | `fileId` path + `?chatId=` | 204 or 503 |

Prefer `/api/v1/library/audiobooks` over the deprecated endpoints. The webhook-based bot (`AudioLibraryBot`) is also registered and handles commands `/list`, `/get`, `/upload`, `/reupload` via Telegram updates.

### Poster — `/api/boosty`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/boosty/post` | **REAL** | `{ "imageDir": "...", "audioDir": "...", "cicleName": "..." }` | `"success"` string |

Uses Playwright to automate posting to Boosty.ru. Matches image/audio pairs by filename. Hardcoded tags: `аудиокнига, дар, магия, фантастика, попаданец, фэнтези, слушать фэнтези, аудиокниги онлайн`.

### Poster — `/api/youtube`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/youtube/upload` | **STUB** | — | Empty string |

No implementation — commented-out placeholder only.

### Text — `/api/text`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/text/description` | **REAL** | — (reads `torrentsPath` from config) | `ExtractedDescription` JSON |

Reads `ParsedBookEntity` records from DB, concatenates descriptions, sends to OpenAI GPT-4o (2500 token limit, 0.3 temperature) for enhancement. Returns combined description.

### AI / OpenAI — `/api/ai/openai`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/ai/openai/image` | **PARTIAL** | `OpenAiResponse` (chat completion JSON) | PNG image + download URL |
| POST | `/api/ai/openai/image/download` | **REAL** | — | Images from `AiTraceService` cache |

The `/image` endpoint extracts text from a chat completion, sends it to DALL-E 3 (1792×1024, vivid style), and downloads the result. Cache is in-memory and lost on restart.

### AI / Ollama — `/api/ai`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/ai/gemma` | **STUB** | Text string body | Empty string |

Sends prompt to local Ollama (`gemma:latest`) but doesn't await the response. Always returns `""`.

### Video — `/api/video`

| Method | Path | Status | Input | Output |
|--------|------|--------|-------|--------|
| POST | `/api/video/create` | **PARTIAL** | PNG image dir, MP4 particle overlay, MP3 audio dir | MP4 video files in `video/` |

Service matches PNG images to MP3 audio by filename and calls `VideoCreation.start()` per pair. FFmpeg overlay composition is incomplete.
