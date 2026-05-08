# Milestone A — Cycle-centric schema implementation plan

> **For agentic workers:** Prefer `superpowers:subagent-driven-development` or `superpowers:executing-plans` task-by-task; use `superpowers:verification-before-completion` before claiming done.

**Implementation priority:** **PostgreSQL first** everywhere (system of record): Flyway, JPA/JDBC, integration tests. **Firestore (Firebase) second** — optional projection/mirror after SQL commits only, never a competing primary; **do not** block Postgres design on Firestore.

**Goal:** Land Flyway DDL for the **cycle-centric** model (`cycles`, `audio_parts`, `video_parts`, `image_parts`, `torrent_tasks`, `scraped_books`, `ai_generated_content`, `image_base64_chunks`, `distribution_records`, `telegram_media_groups`, `telegram_media_items`, `job_logs`), reconcile legacy **`parsed_book`** / **`processing_jobs`**, wire concatenate execution against **`audio_parts`**, and (later / optional) align **Firestore** paths with the **same** logical model—including **Base64 image payloads** in Postgres (inline or chunked per spec §Binary images). **No `facebook_post_data` table.** Optional Firestore collections mirror Postgres only where explicitly wired (**Task 8**).

**Architecture:** **One domain model** in the spec; **Postgres DDL via Flyway** is the authoritative relational encoding. If Firestore is used, it follows **matching** names under **`cycles/{id}/…`** as a **read mirror** after SQL commits — **never** the primary write path for pipeline state.

**Tech Stack:** Spring Boot 3.x, Flyway, PostgreSQL 16, JPA/Hibernate 6, existing executor-based job runner under `jobs/`.

**Derived spec:** [`../specs/2026-05-05-milestone-a-normalized-schema-design.md`](../specs/2026-05-05-milestone-a-normalized-schema-design.md) (**canonical**; replaces ARCHITECTURE §5.2 for this milestone). Implement **§PostgreSQL constraints, indexes, and triggers** in the same Flyway versions as table creation—not as a later cosmetic pass.

---

## File map

| Area | Files |
|------|--------|
| Migrations | … `V202605052200__parsed_book_backfill_scraped_books.sql`; **`V202605052210__library_audiobooks.sql`** (catalog + YouTube/Telegram distribution columns). **`distributedTo`** = **YouTube + Telegram** until other platforms are implemented |
| Jobs | `jobs/domain/ProcessingJobEntity.java`, `ProcessingJobService`, `ConcatenateProcessingJobRunner`, DTOs |
| New JPA | … `ScrapedBookEntity.java`, `CycleEntity`, `AudioPartEntity`, `ImagePartEntity`, `AiGeneratedContentEntity`, `ImageBase64ChunkEntity`, enums … |
| Repositories | … `ScrapedBookRepository`, `CycleRepository`, `AudioPartRepository`, `ImagePartRepository`, `AiGeneratedContentRepository`, `ImageBase64ChunkRepository` |
| REST (cycle / pipeline) | `pipeline/web/CycleV1Controller` — `GET /api/v1/cycles/{id}/scraped-books` |
| REST (library catalog) | **`library/web/AudiobookV1Controller`** — Postgres **`LibraryAudiobookService`** / **`library_audiobooks`** (UUID **`catalog_public_id`** exposed as **`AudiobookDto.id`**) |
| Firebase (optional / lower priority) | Firestore mirror only — **`FirestoreAudiobookService`** invoked from **`LibraryAudiobookService`** when **`firebase.firestore.sync=true`**; never gates REST catalog reads |
| Tests | `integration/FlywayCycleSchemaIT`, `integration/ConcatenateSubmitPostgresIT` (Docker optional via `disabledWithoutDocker`) |

---

## Task 1: Flyway — enums + hub + parts + torrent + scraped

**Files:** `V...__cycle_chunk1_core.sql` (version after latest applied migration)

**Steps:**

- [x] **1.1:** `CREATE TYPE` for all enums listed in spec §“PostgreSQL enums”.
- [x] **1.2:** `CREATE TABLE cycles` with `cycle_identifier` **NOT NULL UNIQUE**, `status` typed `cycle_status`, timestamps.
- [x] **1.3:** `audio_parts`, `video_parts`, `image_parts` with FK `cycle_id REFERENCES cycles(id) ON DELETE CASCADE`; **`image_parts`** includes **`export_path_cache`**, **`inline_base64`**, **`chunk_count`** (default 0), **`mime_type`** per spec §Binary images.
- [x] **1.4:** `torrent_tasks` + unique `torrent_hash`.
- [x] **1.5:** `scraped_books` FK `cycle_id`.
- [x] **1.6:** Per **§PostgreSQL constraints, indexes, and triggers**: **`UNIQUE (cycle_id, part_number)`** on each parts table; **`CHECK (part_number >= 1)`** and duration checks where specified; **`INDEX (status)`** / **`processing_status`** as listed; **`created_at`/`updated_at` `TIMESTAMPTZ NOT NULL DEFAULT now()`**; create **`set_updated_at()`** and **`BEFORE UPDATE`** triggers on **`cycles`**, **`audio_parts`**, **`video_parts`**.

---

## Task 2: Flyway — AI, distribution, Telegram, job_logs

**Files:** `V...__cycle_chunk2_integrations.sql`

**Steps:**

- [x] **2.1:** `ai_generated_content` with **`inline_base64`**, **`chunk_count`**, **`mime_type`**, **`export_path_cache`**, **`prompt`**, **`result`** — see spec §AI generated content + §Binary images; index **`(cycle_id, content_type)`**.
- [x] **2.2:** **`distribution_records`** (FK `audio_part_id REFERENCES audio_parts(id)` **ON DELETE RESTRICT**).
- [x] **2.3:** `telegram_media_groups`, `telegram_media_items` including **`photo_inline_base64`**, **`photo_chunk_count`**, **`photo_mime_type`** (FK item → group; **ON DELETE CASCADE** on items).
- [x] **2.4:** **`image_base64_chunks`** — **after** §2.1 and §2.3 tables exist — three nullable parent FKs, single-parent **`CHECK`**, partial **`UNIQUE`** per spec.
- [x] **2.5:** `job_logs`.
- [x] **2.6:** Apply remaining **§PostgreSQL constraints, indexes, and triggers** for chunk 2 tables (Telegram partial uniques; **`job_logs`** temporal **`CHECK`**; **`image_parts`** **`inline_base64`** / **`chunk_count`** / **`mime_type`** if adjusted in chunk 1 — align with spec).

Do **not** add **`facebook_post_data`** or any Facebook-specific persistence (explicit non-goal per spec).

---

## Task 3: Flyway — `processing_jobs` evolution

**Files:** same chunk as Task 2 or dedicated `V...__processing_jobs_link_audio_parts.sql`

**Steps:**

- [x] **3.1:** Add **`cycle_id BIGINT REFERENCES cycles(id) ON DELETE SET NULL`** to **`processing_jobs`**.
- [x] **3.2:** Add **`audio_part_id BIGINT REFERENCES audio_parts(id) ON DELETE SET NULL`**.
- [x] **3.3:** Normalize **`processing_jobs.status`** to **`job_log_status`** enum **or** align varchar labels with **`PENDING` / `IN_PROGRESS` / `COMPLETED` / `FAILED`** (match **`job_logs`**). Migrate legacy `ACCEPTED`→`PENDING`, `RUNNING`→`IN_PROGRESS`, `SUCCEEDED`→`COMPLETED`.
- [x] **3.4:** Java **`JobStatus`** enum updated to same labels.

---

## Task 4: JPA entities + repositories

**Steps:**

- [x] **4.1:** Map `cycles` and `audio_parts` with `@Column(name = "snake_case")`.
- [x] **4.2:** Extend `ProcessingJobEntity` with `cycleId` / `audioPartId` (or associations).
- [x] **4.3:** `JobLogEntity` + `JobLogRepository`; concatenate runner persists `job_logs` via JPA (replaces JDBC insert).
- [x] **4.4 (slice — image / AI / chunks):** Map `image_parts`, `ai_generated_content`, `image_base64_chunks`; enums **`ImagePartType`** / **`AiContentType`** (PostgreSQL literal names); repositories with ordered queries by `part_number`, `created_at`, `chunk_index`; bulk deletes on chunks by AI row / image part for rewrite flows.
- [x] **4.5:** `ImageBase64ChunkCodec` + **`ImagePayloadService`** (`@ConditionalOnBean(DataSource.class)`): split/join per spec §Binary images; **`saveImagePartPayload`** / **`saveAiGeneratedImagePayload`**; **`readImagePartFullBase64`** / **`readAiGeneratedContentFullBase64`**; tuning via **`pipeline.image.max-inline-base64-chars`** / **`pipeline.image.max-chunk-segment-chars`**.
- [x] **4.6:** **`POST /api/book-cover/generate`** — DB-only contract: **`cycleId`** + **`sourceImagePartId`** (template **`image_parts`** row), optional **`firstPartNumber`** / **`imagePartType`** / overlay **`title`**; **no file paths** in JSON. Response: **`cycleIdentifier`**, **`cycleTitle`**, **`generatedParts[]`** each with **`cycleId`**, **`cycleIdentifier`**, **`cycleTitle`**, **`partNumber`**, **`imagePartId`**; **`export_path_cache`** not used for outputs (null).
- [x] **4.7:** Map **`scraped_books`** (`ScrapedBookEntity` + **`ScrapedBookRepository`**) including **`legacy_parsed_book_id`** after backfill migration **`V202605052200`**.
- [x] **4.8:** Read API **`GET /api/v1/cycles/{cycleId}/scraped-books`** (`CycleV1Controller`, **`ScrapedBookQueryService`**, **`ScrapedBookResponse`**) — **`404`** if **`cycles.id`** missing, **`200`** with JSON array (possibly empty) otherwise.

---

## Task 5: Concatenate service wiring

**Steps:**

- [x] **5.1:** **`AudioConcatenateJobRequest`:** add **`cycleId`** (optional) **or** **`cycleIdentifier`** + **`partNumber`** (default 1). If absent, service creates **`cycles`** row (generated **`cycle_identifier`** = random UUID string).
- [x] **5.2:** Upsert **`audio_parts`** for `(cycle_id, part_number)`; set paths from request; **`processing_status`**: `IN_PROGRESS` → `COMPLETED`/`FAILED`.
- [x] **5.3:** **`ProcessingJobService.submitConcatenate`:** persist **`processing_jobs`** linked to **`audio_part_id`** (and **`cycle_id`**); enqueue runner.
- [x] **5.4:** **`ConcatenateProcessingJobRunner`:** call existing `AudioConcatenatorService`; on completion update **`audio_parts.concatenated_audio_path`**; insert **`job_logs`** row (`job_type`, `cycle_id`, `status`, `message`, timestamps).

---

## Task 6: Backfill `parsed_book` → `scraped_books` (optional same PR)

**Steps:**

- [x] **6.1:** Deterministic **`cycles`** creation per distinct grouping key `COALESCE(source_directory, cycle_name, 'singleton-'||id)` with **`cycle_identifier = 'legacy-grp-' || md5(key)`** (see migration comments).
- [x] **6.2:** Insert **`scraped_books`** from **`parsed_book`** (title, author, reader, year_str→year, genre, cycle_name, cycle_part, description, url→source_url; **`torrent_url`** NULL — not on baseline table).
- [x] **6.3:** Idempotent: **`scraped_books.legacy_parsed_book_id`** + partial unique index; inserts gated with **`WHERE NOT EXISTS`**; **`cycles`** uses **`ON CONFLICT (cycle_identifier) DO NOTHING`**.

---

## Task 7: Tests

- [x] **7.1:** Migration applies on empty DB — **`FlywayCycleSchemaIT`** (`@Testcontainers(disabledWithoutDocker = true)`): Flyway against Postgres 16; asserts **`cycles`**, **`audio_parts`**, **`flyway_schema_history`**.
- [x] **7.2:** Integration test — **`ConcatenateSubmitPostgresIT`**: `@SpringBootTest` + same container guard; **`ProcessingJobService.submitConcatenate`** creates **`cycles`** / **`audio_parts`** / **`processing_jobs`** with matching FKs (`@MockitoBean` **`Firestore`**, **`AudioJobExecutor`**). **Requires Docker** locally/CI to execute; skipped when Docker is unavailable.

---

## Task 8 (optional — after Postgres): Firestore mirror only

**Do not prioritize** until core Postgres + jobs paths are done. Same logical model as SQL — **not** a second write authority.

- [ ] **8.1:** Map existing **`firebase.firestore.collection`** usage to **`cycles`** root (or introduce `cycles` alongside legacy during migration with a feature flag).
- [ ] **8.2:** On successful SQL writes for **`cycles`** / **`audio_parts`** (concatenate path), **optional async** projection to Firestore documents using paths from spec §Firestore mirror layout and **camelCase** field names matching logical ER.
- [ ] **8.3:** Stop introducing new fields on legacy **`audiobooks`** documents unless they map 1:1 to this ER; prefer **new reads** from **`cycles/{cycleDocId}`** tree when mirror exists.

---

## Milestone B — pipeline tables without full app wiring (slice-by-slice)

DDL from Milestone A already includes **`distribution_records`**, **`telegram_media_groups`** / **`telegram_media_items`**, and **`torrent_tasks`** as part of the cycle-centric model. **Application wiring** (services, REST, schedulers, Telegram paths, torrent runner state in SQL) is **not** required to complete Milestone A.

Pick **one vertical slice at a time** when extending Milestone B, for example:

1. **Torrent** — persist qBittorrent task lifecycle into **`torrent_tasks`** and link to **`cycles`** / **`audio_parts`** where applicable.
2. **Distribution** — insert **`distribution_records`** when Boosty/YouTube/Telegram posting succeeds; align with optional Firestore mirror.
3. **Telegram structured media** — replace or complement legacy JSON blobs with **`telegram_media_groups`** / **`telegram_media_items`** where the bot manages grouped uploads.

Avoid big-bang refactors; keep Postgres authoritative and mirror to Firestore only after SQL commits when opted in.

---

## Self-review vs spec

| Spec section | Tasks |
|--------------|--------|
| ER / enums / constraints / triggers (chunk 1) | 1 |
| AI / distribution / Telegram / chunks / `job_logs` (chunk 2) | 2 |
| Legacy `processing_jobs` | 3 |
| JPA (minimal) | 4 |
| Concatenate → `audio_parts` | 5 |
| Backfill (`parsed_book` → `scraped_books`) | 6 |
| Tests | 7 |
| Firestore mirror (optional) | 8 |

---

## Execution options

**Plan file:** `docs/superpowers/plans/2026-05-05-milestone-a-normalized-schema-implementation-plan.md`

1. **Subagent-driven** — e.g. Flyway Tasks 1–3 vs Java 4–5 first; **Task 8 (Firestore)** only when Postgres work is stable.  
2. **Inline** — Tasks **1→7** in order (skip optional Task 6 if backfill deferred); **Task 8** optional last.

**Focus:** PostgreSQL + Flyway + JPA/JDBC — **not** Firestore-first.
