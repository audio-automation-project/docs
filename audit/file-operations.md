# File Operations — What the System Handles

---

## Directory Layout

All paths are relative to `AUDIO_BASE_PATH` (configured via env var; default `H:/Projects/Audio-library-System/audio-library-automation-bot/data`). Directories are auto-created by `ProperityConfig` on first access.

```
{AUDIO_BASE_PATH}/
  audio/        source MP3 files (input for compress, trim, concatenate)
  audioC/       compressed MP3 output (128 kbps)
  video/        MP4 video output
  image/        generated PNG book covers
  torrents/     torrent staging area (read by text extraction + torrent upload)
  metadata/     concatenation metadata JSON files
  temp/         temporary processing files
```

---

## Input — Files the System Ingests

| Format | Source | Used By | Notes |
|--------|--------|---------|-------|
| `.mp3` | `audio/` directory | Audio: compress, trim, concatenate, duration | Must be named consistently for pair-matching |
| `.png` | `image/` directory | Video: creation (image frames) | Matched to MP3 by filename (minus extension) |
| `.mp4` | Configured path | Video: creation (particle overlay) | Used as overlay layer composited with image+audio |
| Base image file | Any path (request body) | Image: book cover generation | Any format readable by Java ImageIO |
| `audiobooks.json` | `metadata/` or classpath | Library: one-shot import | Migrates legacy JSON records to PostgreSQL |
| Torrent files | `torrents/` directory | Torrent: upload to qBittorrent | Read from staging dir, uploaded via qBittorrent Web API |
| `AudiobookDto` JSON | HTTP request body | Library: POST/PUT endpoints | Creates/updates records in PostgreSQL |
| Search query string | HTTP query param | Search: `/api/search?query=` | Sent to LLM endpoint, returns URL list |

---

## Produced — Files the System Creates

| Format | Written To | Produced By | Notes |
|--------|-----------|-------------|-------|
| `.mp3` (compressed) | `audioC/` | Audio: `/api/audio/compress` | 128 kbps via `libmp3lame` |
| `.mp3` (concatenated) | `audio/` or `audioC/` | Audio: `/api/audio/concatenate` | Single file from all MP3s in input dir |
| `.mp3` (trimmed) | Same dir as source | Audio: `/api/audio/trim` | Cuts audio from `startSeconds` to end |
| `.png` (book cover) | `image/` | Image: `/api/book-cover/generate` | Named `1.png`, `2.png`, … up to `count` |
| `.png` (DALL-E) | Configured download path | AI: `/api/ai/openai/image` | Downloaded from OpenAI URL, stored locally |
| `.mp4` (video) | `video/` | Video: `/api/video/create` | Partial — full FFmpeg composition incomplete |
| Metadata JSON | `metadata/` | Audio: concatenate | Stores concatenation run metadata via `AudioDescriptionPersistenceService` |

---

## Distributed — Files Sent to External Systems

| Destination | What | Mechanism | Endpoint |
|-------------|------|-----------|----------|
| **Boosty.ru** | PNG image + MP3 audio (as post) | Playwright browser automation | `POST /api/boosty/post` |
| **Telegram** | Audio files (reupload by fileId) | Telegram Bot API | `POST /api/bot/reupload/{fileId}` |
| **Firestore** | `AudiobookDto` metadata | Firebase Admin SDK (async) | Triggered automatically after any Library write, if `firebase.firestore.sync=true` |
| **OpenAI** | Text prompt | REST HTTP | Used internally by `/api/ai/openai/image` and `/api/text/description` |
| **qBittorrent** | Torrent files | qBittorrent Web API | `POST /api/torrent/upload` |
| **YouTube** | — | — | `/api/youtube/upload` is a stub — nothing is sent |

---

## Read-only Scraping — External Sources

| Source | What is Fetched | Stored As | Endpoint |
|--------|----------------|-----------|----------|
| `baza-knig.ru` | Book title, description, URLs | `parsed_book` PostgreSQL table | `POST /api/parser/parse` |
| Google (via LLM proxy) | Search result URLs | Returned in `SearchInformation` response | `GET /api/search` |

---

## File Lifecycle — Audio Processing Pipeline

The typical end-to-end flow for processing a new audiobook:

```
1. Torrent download completes
   └─ Files land in torrents/ staging

2. POST /api/text/description
   └─ Reads parsed_book records
   └─ Sends to GPT-4o → returns enhanced description

3. POST /api/audio/compress
   └─ audio/ → audioC/ (128 kbps MP3s)

4. POST /api/audio/concatenate
   └─ audioC/ → single MP3 + metadata JSON

5. POST /api/audio/duration
   └─ Returns duration string for metadata

6. POST /api/book-cover/generate
   └─ Base image → N × PNG covers in image/

7. POST /api/video/create  [PARTIAL]
   └─ image/ + audio/ + particle.mp4 → video/

8. POST /api/boosty/post
   └─ image/ + audioC/ → Boosty post (Playwright)
```

---

## Static code scan — file / JSON touchpoints (`main` Java, snapshot 2026-05-05)

Inventory for **`C1` hardening** (replace whole-file patterns with SQL bounded writes, capped reads, or facades). Re-run search when refactoring.

| Path | Pattern | Risk / note | Suggested direction |
|------|---------|-------------|---------------------|
| `torrent/service/TorrentService.java` | Commented **`FileReader` / `description.json`** / Gson list | Was high risk (full-file read/write); currently dead code | Delete block or migrate to `processing_jobs` / SQL if revived |
| `torrent/service/QBittorrentService.java` | **`gson.fromJson`** on API responses | Bounded HTTP payload | Keep; add size guard at HTTP layer if needed |
| `poster/boosty/service/Poster.java` | **`Files.readAllBytes` → `cookies.json`**, Gson / ObjectMapper | Unbounded disk read of auth blob | Prefer env-mounted secret path + size cap |
| `video/service/VideoCreation.java`, `QBittorrentController` | **`Files`** create/dir scan, Gson logging | Operational I/O | Document max directory size; streaming N/A |
| `audio/*/AudioConcatenatorService` … | **`Files.createTempFile`**, Writers | Controlled temp concat list | Covered by tests; keep bounded pool for jobs |
| `telegram/component/AudioLibraryBot.java` | **`FileOutputStream`** download temp | Telegram file staging | Lifecycle: delete after upload |
| `jobs/*` | **ObjectMapper** on job payload strings | Small JSON payloads | OK for orchestration |

`ProperityConfig` / `AUDIO_BASE_PATH` narrative above may be historical if path config moved — reconcile with [`repository-map`](../architecture/repository-map.md) and bot `README` when editing deploy docs.
