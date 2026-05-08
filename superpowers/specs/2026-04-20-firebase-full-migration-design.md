# Design Spec: Full Firebase Migration

**Date:** 2026-04-20
**Scope:** Replace PostgreSQL + local filesystem with Firestore as the single platform for metadata, status, and image storage. Audio/video files are ephemeral — processed locally and deleted after distribution.
**Approach:** Big Bang — full cutover, system offline during migration.

> **Reconciliation (current codebase):** The bot stayed **Postgres-first**. The operational catalog lives in **`library_audiobooks`** (Flyway `V202605052210__library_audiobooks.sql`). **`distributedTo`** in Java/API is **`youtubeVideoId`** + **`telegramFileId`** only — treat **`boostyUrl`** in this historical document as superseded.

---

## Context

The Audio Library System currently depends on:
- **PostgreSQL** for all metadata persistence (audiobooks, parsed books, media groups)
- **Local filesystem** (Windows paths via `ProperityConfig`) for all file I/O (audio, images, video, temp)
- **Flyway** for database migrations
- **Spring Data JPA / Hibernate** for ORM

These create an infrastructure dependency: the system only runs on a machine with a PostgreSQL server and specific local paths.

**The key architectural insight:** This system is a **processing pipeline, not a file archive**. Audio files (up to 5 GB per book) are downloaded from torrents, processed with FFmpeg, distributed to Boosty/Telegram/YouTube, and then deleted. They do not need to be stored long-term anywhere. Only metadata, status, and small image assets need persistence.

**After the migration:**

- **Firestore** holds all metadata, job status, and chunked image data (no PostgreSQL, no Firebase Storage)
- **Spring Boot** is a stateless processing engine — any machine with a service account JSON can run it
- **React frontend** reads job status from Firestore in real time
- **All audio/video files** are temp-only: downloaded → processed → distributed → deleted

---

## Section 1: Firestore Data Model

Firebase Storage is **not used**. All persistent data lives in Firestore collections. Images are stored as chunked base64 across two collections to stay within Firestore's 1 MB per-document limit.

### Core collections

```
/audiobooks/{id}
  fileId, title, author, narrator, originalTitle, part,
  sourceLink, duration, description, uploadDate,
  coverImageIds: ["coverId1", "coverId2"],   ← refs to /image_store
  distributedTo: {
    youtubeVideoId: string | null,
    telegramFileId: string | null
  },
  processingStatus: "IDLE" | "PROCESSING" | "DISTRIBUTED" | "FAILED",
  createdAt, updatedAt

/parsed_books/{id}
  title, description, sourceDirectory, url, createdAt

/media_groups/{id}
  name, items: [{ audiobookId: string, order: int }]
```

### Image storage — chunked base64

Images (book covers, DALL-E outputs) are compressed then split into ≤700 KB base64 chunks, each stored as a separate Firestore document. This avoids the 1 MB document limit while keeping all data in Firestore.

```
/image_store/{imageId}                  ← metadata document
  type: "cover" | "ai_generated"
  bookId: string
  coverIndex: int | null                ← for type="cover"
  mimeType: "image/jpeg"
  totalChunks: int
  originalSizeBytes: int
  createdAt: timestamp

/image_chunks/{chunkId}                 ← one doc per chunk
  imageId: string
  chunkIndex: int
  data: string                          ← base64 chunk, max ~700 KB
```

**Reassembly:** query `/image_chunks` where `imageId == X` ordered by `chunkIndex`, concatenate `data` fields, decode base64.

**Compression rule before storage:** All images must be compressed to JPEG (quality 80) and resized to a max width/height of 1400 px before chunking. This keeps most covers to 1 chunk (150–350 KB). DALL-E 1792×1024 PNGs resize to ~250 KB JPEG = 1 chunk.

### Job tracking and AI logging

```
/job_logs/{jobId}
  type: "COMPRESS" | "CONCATENATE" | "COVER_GENERATE" | "DISTRIBUTE" | "TRIM"
  status: "STARTED" | "COMPLETED" | "FAILED"
  audiobookId: string
  message: string
  errorDetail: string | null
  startedAt: timestamp
  finishedAt: timestamp | null

/ai_traces/{id}
  prompt: string
  imageId: string                       ← ref to /image_store
  generatedAt: timestamp

/ai_integration_log/{id}
  provider: "openai" | "ollama"
  model: string                         ← "gpt-4o", "dall-e-3", "gemma:latest"
  prompt: string
  tokensUsed: int | null
  costUsd: float | null
  calledAt: timestamp
  success: boolean
```

---

## Section 2: Processing Pipeline (Ephemeral Files)

**Firebase Storage is not used.** All file processing happens in a local temp directory, configured via:

```properties
processing.temp.dir=${PROCESSING_TEMP_DIR:${java.io.tmpdir}/audio-pipeline}
```

Defaults to the OS temp directory. Any machine, any path. Overridable per deployment.

### Temp directory structure during a job

```
{PROCESSING_TEMP_DIR}/
  {jobId}/
    source/        downloaded MP3s (from torrent, ~5 GB)
    compressed/    128 kbps output (~2 GB)
    final.mp3      concatenated output
    covers/        generated PNG/JPEG covers
    video/         rendered MP4 (if applicable)
```

The `{jobId}/` subtree is created at job start and **deleted entirely** after the job completes or fails.

### Processing contract

```
1. Job starts  → create {PROCESSING_TEMP_DIR}/{jobId}/
               → write STARTED to /job_logs/{jobId}

2. Download    → torrent files land in source/ (via qBittorrent)

3. Process     → FFmpeg runs on local temp files (unchanged)
               → covers generated into covers/

4. Compress    → upload cover JPEGs to Firestore as chunked base64

5. Distribute  → Boosty: Playwright reads from covers/, audio from compressed/
               → Telegram: bot sends compressed/ files directly
               → YouTube: (stub)

6. Update DB   → write **`distributedTo.youtubeVideoId`** / **`telegramFileId`** (current catalog shape) to Postgres **`library_audiobooks`** and optionally mirror `/audiobooks/{id}`
               → write DISTRIBUTED to /job_logs/{jobId}

7. Cleanup     → delete entire {PROCESSING_TEMP_DIR}/{jobId}/ tree
```

FFmpeg and Playwright never interact with Firestore or any network storage — they always work on local temp files.

---

## Section 3: Backend Code Changes

### Dependencies — `pom.xml`

**Remove:**
- `spring-boot-starter-data-jpa`
- `postgresql` JDBC driver
- `flyway-core`
- `h2` (test scope)
- `google-cloud-storage` (not needed — no Firebase Storage)

**No new dependencies** — `firebase-admin` is already present.

### New classes — `firebase/` package

| Class | Replaces | Responsibility |
|-------|---------|----------------|
| `FirestoreAudiobookService` | `AudiobookLibraryService` | full CRUD on `/audiobooks` collection |
| `FirestoreImageService` | `ProperityConfig` (image path) + local write | compress image → chunk → write to `/image_store` + `/image_chunks`; reassemble on read |
| `FirestoreJobLogService` | *(none — new)* | writes job start/finish/error to `/job_logs/` |
| `FirestoreAiLogService` | `AiTraceService` (in-memory) | logs AI calls to `/ai_integration_log/`, traces to `/ai_traces/` |

### Classes removed

- `AudiobookEntity`, `ParsedBookEntity`, `MediaGroupEntity` — JPA entity classes
- `AudiobookRepository` — Spring Data JPA interface
- `AudiobookLibraryService` — JPA-backed service
- `JsonAudiobookImportService` — one-shot JSON import, no longer needed
- `ProperityConfig` (`file/ProperityConfig.java`) — replaced by `processing.temp.dir` property + `FirestoreImageService`
- All Flyway SQL files in `db/migration/`

### Classes modified

| Class | Change |
|-------|--------|
| `AudioService` | Replace `ProperityConfig` path refs with `processing.temp.dir` resolution; write job log via `FirestoreJobLogService` |
| `VideoService` | Same temp dir pattern; write job log |
| `ImageService` (book cover) | Generate JPEG to temp dir → call `FirestoreImageService.store()` → delete local file |
| `TextService` | Read `parsed_books` from Firestore instead of `ParsedBookEntity` JPA |
| `AiTraceService` | Backed by Firestore `/ai_traces/` instead of in-memory `Map`; image stored via `FirestoreImageService` |
| `BoostyService` | *(Historical doc)* Today’s mirror uses `setDistributedTo(id, youtubeVideoId, telegramFileId)` — Boosty URLs are **not** stored on **`library_audiobooks`** |
| `TelegramService` | After successful send: update **`telegramFileId`** on the catalog (Postgres + optional Firestore mirror) |
| `AudiobookV1Controller` | Delegates to `FirestoreAudiobookService` instead of `AudiobookLibraryService` |
| `RestCallService` | Move hardcoded OpenAI API key to `@Value("${openai.api.key}")` |
| `SearchService` | Move hardcoded Bearer token to `@Value("${search.api.token}")` |
| `FirebaseConfig` | Remove `@ConditionalOnProperty` — Firebase is always enabled now |

### `application.properties` changes

**Remove:**
```properties
DATABASE_URL, DATABASE_USER, DATABASE_PASSWORD
spring.datasource.*
spring.jpa.*
spring.flyway.*
AUDIO_BASE_PATH and all path.default.* properties
app.migration.import-json
app.migration.import-media-groups
```

**Add / update:**
```properties
firebase.enabled=true
firebase.firestore.sync=true
GOOGLE_APPLICATION_CREDENTIALS=${FIREBASE_SERVICE_ACCOUNT_PATH}

processing.temp.dir=${PROCESSING_TEMP_DIR:${java.io.tmpdir}/audio-pipeline}

openai.api.key=${OPENAI_API_KEY}
search.api.token=${SEARCH_API_TOKEN}
```

---

## Section 4: Frontend Changes

The React frontend remains an **operational dashboard** — no library browser.

**New npm dependency:** `firebase` (Firestore JS SDK only — no Storage SDK needed)

**New `src/firebase.ts`:**
```typescript
import { initializeApp } from 'firebase/app';
import { getFirestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  // remaining fields from Firebase console
};
export const app = initializeApp(firebaseConfig);
export const db = getFirestore(app);
```

**`.env` additions:**

```properties
VITE_FIREBASE_API_KEY=...
VITE_FIREBASE_PROJECT_ID=...
```

**`src/api/client.ts` changes:**

- Add: Firestore `onSnapshot` listener on `/job_logs` ordered by `startedAt desc`, limit 20 — live job status feed
- Keep: `GET /actuator/health` polling for backend + Python service health
- Remove: any REST polling for audiobook data

**`App.tsx` changes:**

- Job status panel: reads from Firestore real-time listener, auto-updates as backend writes
- Health check panels: unchanged
- Processing triggers: still call backend REST endpoints (`POST /api/audio/compress`, etc.)

---

## Section 5: One-Time Migration Execution

Execute in order. System is offline for the duration.

1. **Firebase setup**
   - Reuse existing `audio-library-system` Firebase project (already in `.firebaserc`)
   - Enable Firestore in Native Mode
   - **Do NOT enable Firebase Storage** (not needed)
   - Generate service account JSON, set `GOOGLE_APPLICATION_CREDENTIALS`

2. **Export PostgreSQL data**
   - Dump `audiobook`, `parsed_book`, `media_group` tables to JSON files

3. **Run Firestore import script**
   - One-shot Python script using `firebase-admin`: reads JSON dumps, creates Firestore documents
   - For each audiobook: set `processingStatus = "DISTRIBUTED"` if it has a known Boosty URL, else `"IDLE"`
   - For existing cover images: compress + chunk each PNG → write to `/image_store` + `/image_chunks`; store `coverImageIds` on the audiobook document
   - `audioFilePath`/`coverImagePath` local path fields are dropped (not migrated — files will be reprocessed if needed)

4. **Apply backend code changes** (Section 3)

5. **Apply frontend code changes** (Section 4)

6. **Update `application.properties`** (Section 3)

7. **End-to-end test**
   - Trigger `POST /api/audio/compress` with a test book
   - Verify temp dir is created, populated, then cleaned up
   - Verify `/job_logs` in Firestore updates in real time
   - Verify frontend shows live job status via Firestore listener
   - Verify book cover appears in Firestore as chunked base64

8. **Decommission**
   - Stop PostgreSQL server
   - Delete local `audio/`, `image/`, `video/`, `audioC/` directories
   - Remove `spring-boot-starter-data-jpa`, `postgresql`, `flyway` from dev machine if not used elsewhere

---

## What Stays Unchanged

- FFmpeg processing logic (`AudioService`, `VideoService`, `ExecutionService`)
- Playwright automation (`BoostyService`, `BookParserService`)
- Telegram bot (`AudioLibraryBot`)
- OpenAI/Ollama integration logic (only key injection changes)
- qBittorrent integration (`TorrentService`)
- All REST endpoint paths (`/api/audio/*`, `/api/v1/library/audiobooks`, etc.)
- Python ML service (`audio-silence-service`) — no changes needed
- CI/CD workflows in `audio-platform-cicd/`
