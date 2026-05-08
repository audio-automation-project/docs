# File System Isolation & Lifecycle Design

**Date:** 2026-05-07
**Status:** Approved — ready for implementation
**Scope:** `audio-library-automation-bot` — all services that read/write files

---

## Problem Statement

The system processes 10+ books (cycles) in parallel. Five services construct file paths independently using hardcoded strings or caller-supplied parameters, causing silent file collisions when two jobs run concurrently. Two of these are critical: `VideoCreation` and `RestCallService` write to shared flat paths with no job-scoping, meaning concurrent jobs overwrite each other's intermediate files. Additionally, no lifecycle mechanism exists to safely clean up files after a book is fully distributed — files either linger indefinitely or risk being deleted before uploads are confirmed.

---

## Approach: Cycle + Part Directory Isolation (Option B)

Every file lives in a path derived from `cycleId` + `partId` + `jobId`. A single new service (`CycleFileLayout`) owns all path construction. No other service constructs paths directly.

---

## Section 1: Directory Structure

```
{processing.temp.dir}/
  {cycleId}/
    part_{partId}/
      source/              ← raw .mp3 segments from torrent download
      compressed/          ← after Demucs noise removal
      concat/              ← after concatenation → part_{partId}.mp3
    covers/                ← one cover image per cycle
      cover.png            ← also persisted as Base64 to DB
    video/
      part_{partId}.mp4    ← final video per part
    tmp/                   ← short-lived intermediate files only
      {jobId}_pre_rendered.mp4
      {jobId}_pre_scaled.jpg
      {jobId}_download.png
```

**Rules:**
1. No service may call `Path.of()` or string concatenation with a base dir. All paths come from `CycleFileLayout`.
2. Any intermediate file (not a final deliverable) must use `tmpFile(cycleId, jobId, suffix)` and be deleted in a `finally` block after use.
3. `CycleFileLayout` calls `Files.createDirectories()` before returning any path. Callers never mkdir themselves.

---

## Section 2: CycleFileLayout Service (New)

**Package:** `pipeline/service/`
**Spring bean:** `@Service`

```java
public class CycleFileLayout {
    // @Value("${processing.temp.dir:...}")
    private final String base;

    public Path cycleDir(long cycleId)
    public Path partSourceDir(long cycleId, long partId)
    public Path partCompressedDir(long cycleId, long partId)
    public Path partConcatFile(long cycleId, long partId)         // → concat/part_{partId}.mp3
    public Path coverFile(long cycleId)                           // → covers/cover.png
    public Path videoFile(long cycleId, long partId)              // → video/part_{partId}.mp4
    public Path tmpFile(long cycleId, UUID jobId, String suffix)  // → tmp/{jobId}_{suffix}
    public void deleteCycleDir(long cycleId)                      // walk-and-delete, used by scheduler
    public void deleteTmpFile(Path p)                             // safe delete, swallows missing-file
}
```

`ProcessingTempDirService` is deprecated — it delegates to `CycleFileLayout` temporarily so existing tests don't break, and is removed in a follow-up.

---

## Section 3: Fixes to Existing Services

### 3.1 VideoCreation.java — CRITICAL

**Problem:** Two hardcoded shared paths in `src/main/resources/temp/` — all concurrent video jobs overwrite the same intermediate files.

**Fix:** Method gains `cycleId` and `jobId` parameters. Intermediate paths routed through `CycleFileLayout.tmpFile()`. Existing `finally` block already calls `Files.deleteIfExists()` — no change needed there.

```java
// Before
Path preRendered = Paths.get("src/main/resources/temp/pre_rendered_segment.mp4");
Path preScaled   = Paths.get("src/main/resources/temp/pre_scaled_image.jpg");

// After
Path preRendered = layout.tmpFile(cycleId, jobId, "pre_rendered.mp4");
Path preScaled   = layout.tmpFile(cycleId, jobId, "pre_scaled.jpg");
```

### 3.2 RestCallService.java — CRITICAL + FILE LEAK

**Problem:** All image downloads write to `{tempDir}/base.png` — concurrent downloads overwrite each other. File is never deleted after use.

**Fix:** `downloadImage()` gains `cycleId` + `jobId` parameters and returns the `Path`. The **caller** owns cleanup — it wraps usage in a `try/finally` and calls `layout.deleteTmpFile(path)` after use. `downloadImage()` must not delete in its own `finally` because the file would be gone before the caller can read it (Java executes `finally` before the return value is used by the caller's stack frame).

```java
// Before
String savePath = tempDir + "/base.png";
saveImageLocally(response.getBody(), savePath);
return savePath; // never deleted

// After — downloadImage() just creates and returns
Path tmp = layout.tmpFile(cycleId, jobId, "download.png");
saveImageLocally(response.getBody(), tmp);
return tmp; // caller is responsible for deletion

// Caller pattern (e.g. BookCoverService, VideoCreation):
Path img = restCallService.downloadImage(cycleId, jobId);
try {
    useImage(img);
} finally {
    layout.deleteTmpFile(img);
}
```

### 3.3 QBittorrentController.java + TorrentService.java — MEDIUM-HIGH

**Problem:** All torrent downloads land in flat shared `{tempDir}/torrents/` and `{tempDir}/audio/` directories. Two books with identically named part files collide.

**Fix:** Torrent download endpoint requires `cycleId` + `partId` in the request body. Downloads go to `layout.partSourceDir(cycleId, partId)`. Flat `/torrents` and `/audio` dirs are no longer created.

### 3.4 ConcatenateProcessingJobRunner.java — MEDIUM

**Problem:** `inputDirectory` and `outputDirectory` are raw strings from the HTTP caller stored in `input_payload`. A bad caller can point two jobs at the same folder.

**Fix:** Paths are derived from `cycleId` + `audioPartId` already on the job entity. The payload no longer carries file paths.

```java
// Before
AudioConcatenateJobRequest req = mapper.readValue(job.getInputPayload(), ...);
// uses req.getInputDirectory() / req.getOutputDirectory()

// After
Path inputDir  = layout.partSourceDir(job.getCycleId(), job.getAudioPartId());
Path outputDir = layout.partConcatFile(job.getCycleId(), job.getAudioPartId()).getParent();
```

### 3.5 Controllers accepting raw path parameters

`AudioController` (compress, concatenate, trim) and `VideoController` currently accept `inputDirectory` / `outputDirectory` as raw request strings. These endpoints are deprecated for internal pipeline use — they remain for manual/debug calls only and add a note in the response that they bypass isolation guarantees.

---

## Section 4: Lifecycle State Machine

### States

```
ACTIVE → UPLOADS_CONFIRMED → APPROVED_FOR_DELETE → FILES_DELETED
  ↓ (any stage)
FAILED
```

| State | Meaning |
|---|---|
| `ACTIVE` | Cycle is being processed or awaiting uploads |
| `UPLOADS_CONFIRMED` | All configured platforms reported upload success |
| `APPROVED_FOR_DELETE` | Human clicked Approve in dashboard |
| `FILES_DELETED` | Disk cleared; DB rows intact |
| `FAILED` | A critical step failed; files kept for inspection |

### DB Changes

**Migration 1 — `V{n}__cycle_file_lifecycle.sql`:**
```sql
ALTER TABLE cycles
  ADD COLUMN file_lifecycle VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',
  ADD COLUMN target_platforms TEXT NOT NULL DEFAULT 'YOUTUBE,TELEGRAM';
  -- comma-separated list of platforms this cycle must upload to before
  -- UPLOADS_CONFIRMED is possible. Set when cycle is created.
  -- Example values: 'YOUTUBE,TELEGRAM' | 'YOUTUBE,PATREON,TWITCH,TELEGRAM'

CREATE INDEX idx_cycles_lifecycle
  ON cycles(file_lifecycle)
  WHERE file_lifecycle IN ('APPROVED_FOR_DELETE');
```

**Migration 2 — `V{n+1}__audio_stages_unique.sql`:**
```sql
ALTER TABLE audio_stages
  ADD CONSTRAINT uq_audio_stages_part_stage
  UNIQUE (audio_part_id, stage);
```

The partial index makes the cleanup scheduler's query fast — it only scans cycles awaiting deletion.

### New DB Table: `upload_results`

```sql
CREATE TABLE upload_results (
    id          BIGSERIAL PRIMARY KEY,
    cycle_id    BIGINT NOT NULL REFERENCES cycles(id),
    platform    VARCHAR(32) NOT NULL,  -- YOUTUBE | PATREON | TWITCH | TELEGRAM
    status      VARCHAR(16) NOT NULL,  -- SUCCESS | FAILED
    external_id VARCHAR(256),          -- YouTube video ID, Telegram file ID, etc.
    completed_at TIMESTAMPTZ NOT NULL
);
CREATE UNIQUE INDEX uq_upload_results_cycle_platform
  ON upload_results(cycle_id, platform);
```

### Transition Rules

- **`ACTIVE` → created automatically** when a cycle is first created. `CycleFileLayout.cycleDir()` is called at this point to create the directory tree.
- **`ACTIVE` → `UPLOADS_CONFIRMED`** — triggered by `UploadConfirmationService.check(cycleId)` after each upload job completes. Splits `cycles.target_platforms` by comma, checks that every platform in that set has a `SUCCESS` row in `upload_results` for this cycle.
- **`UPLOADS_CONFIRMED` → `APPROVED_FOR_DELETE`** — triggered exclusively by `POST /api/v1/cycles/{id}/approve-delete`. Backend guard: throws `IllegalStateException` if current state is not `UPLOADS_CONFIRMED`. Cannot be triggered by any automated process.
- **`APPROVED_FOR_DELETE` → `FILES_DELETED`** — triggered by `CycleCleanupScheduler` (hourly). Calls `CycleFileLayout.deleteCycleDir()`, then updates state.

---

## Section 5: New Services

### CycleLifecycleService (New)
The only service allowed to write `file_lifecycle`. Handles all state transitions with guards.

### UploadConfirmationService (New)
Called after every upload job completes. Queries `upload_results` for the cycle — if all configured platforms have `SUCCESS`, calls `CycleLifecycleService.onAllUploadsConfirmed()`.

### CycleCleanupScheduler (New)
`@Scheduled(fixedDelay = 3_600_000)` — runs every hour.

1. **Delete approved cycles:** finds all `APPROVED_FOR_DELETE` cycles, calls `layout.deleteCycleDir()`, transitions to `FILES_DELETED`.
2. **Sweep stale tmp/ files:** for all `ACTIVE` cycles, deletes any file in `tmp/` older than 2 hours. Safety net for crashed jobs.

Both operations are idempotent — running twice on the same cycle is safe.

### REST: CycleV1Controller (Extended)
One new endpoint on the existing controller:
```java
@PostMapping("/{id}/approve-delete")
@ResponseStatus(NO_CONTENT)
public void approveForDeletion(@PathVariable long id) {
    lifecycleService.approveForDeletion(id);
}
```
`CycleResponse` gains `fileLifecycle` and `filesDeletedAt` (nullable) fields.

---

## Section 6: What Gets Deleted vs Kept

| Item | On `FILES_DELETED` |
|---|---|
| Entire `{cycleId}/` directory tree | **Deleted** |
| `cycles`, `audio_parts`, `audio_stages` DB rows | **Kept** |
| `job_logs`, `upload_results` DB rows | **Kept** |
| Cover image (Base64 in `image_parts`) | **Kept** |
| Upload IDs (YouTube video ID, Telegram file ID) | **Kept** in `library_audiobooks` |

The DB is the permanent audit trail. Only disk files are released.

---

## Section 7: Immutable Rules Summary

| # | Rule |
|---|---|
| 1 | No ad-hoc path construction — all paths from `CycleFileLayout` |
| 2 | `tmp/` files are always `jobId`-scoped and deleted in a `finally` block |
| 3 | `CycleFileLayout` creates parent directories — callers never mkdir |
| 4 | Human approval is the only gate to deletion — no automated process skips it |
| 5 | `tmp/` is cleaned inline (primary) + hourly sweep (safety net) |
| 6 | Upload confirmation tracked in `upload_results` table, not inferred from job status |
| 7 | Cleanup scheduler is idempotent — safe to run multiple times |
| 8 | No cascading DB deletes — `FILES_DELETED` means disk only |

---

## Implementation Order

1. `CycleFileLayout` service + tests
2. Flyway migration 1 (`file_lifecycle` column) + migration 2 (`audio_stages` unique constraint)
3. Fix `VideoCreation` + `RestCallService` (critical — unblock parallel video jobs)
4. Fix `TorrentService` + `QBittorrentController` (cycle+part scoped download dirs)
5. Fix `ConcatenateProcessingJobRunner` (derive paths from job metadata)
6. `upload_results` table migration + `UploadConfirmationService`
7. `CycleLifecycleService` + `CycleCleanupScheduler`
8. `POST /cycles/{id}/approve-delete` endpoint + `CycleResponse` fields
9. Deprecate `ProcessingTempDirService` (delegate to `CycleFileLayout`)
10. Frontend: lifecycle panel + "Approve Delete" button on cycle detail page
