# Pipeline Orchestrator — RabbitMQ-Driven Production Pipeline

**Date:** 2026-05-08  
**Status:** Design approved, implementation planned across 3 sub-plans.

---

## Goal

Replace ad-hoc service-to-service calls with a fully job-queue-driven production pipeline.
Each processing step is a `job_queue` row dispatched via RabbitMQ. A `PipelineOrchestrator`
advances the pipeline by querying DB state — it never calls business services directly.

---

## Architecture

### Three coordinated pipelines (key unit: `cycle_id`)

| Pipeline | Purpose | Tables written |
|---|---|---|
| **A — Acquisition** | Scrape + torrent download | `book`, `book_chapter`, `torrent_task` |
| **B — Production** | Audio → Image → Video | `audio_part`, `audio_stage`, `image_asset`, `video_part`, `video_stage`, `ai_content` |
| **C — Distribution** | Upload → record outcomes | `distribution`, `distribution_result`, `library_audiobook` |

All async work goes through `job_queue`. Pipeline B is the primary focus.

### Job type → FK ownership (one FK per job type)

| JobType | Owner FK | Stage produced |
|---|---|---|
| `AUDIO_NOISE_REMOVE` | `audio_part_id` | `audio_stage.NOISE_REMOVED` |
| `AUDIO_TRIM` | `audio_part_id` | `audio_stage.TRIMMED` |
| `AUDIO_CONCATENATE` | `audio_part_id` (full-book part) | `audio_stage.CONCATENATED` |
| `AUDIO_COMPRESS` | `audio_part_id` | `audio_stage.COMPRESSED` |
| `AUDIO_DURATION` | `audio_part_id` | writes `audio_part.duration_seconds` |
| `IMAGE_GENERATE` | `image_asset_id` | fills `image_asset` |
| `VIDEO_RENDER` | `video_part_id` | `video_stage.PRE_RENDERED` or `FINAL` |
| `DISTRIBUTE` | `distribution_id` | `distribution_result` rows per platform |
| `CLEANUP_CYCLE` | `cycle_id` only | deletes filesystem files |

### Phase gates (orchestrator DB queries)

```
1. NOISE_REMOVE all parts complete  →  enqueue AUDIO_TRIM per part
2. TRIM all parts complete          →  enqueue AUDIO_DURATION per part + AUDIO_CONCATENATE
3. CONCATENATE complete             →  enqueue AUDIO_COMPRESS
4. COMPRESS complete                →  enqueue IMAGE_GENERATE (parallel) + when image ready → VIDEO_RENDER
5. VIDEO_RENDER complete            →  enqueue DISTRIBUTE
6. DISTRIBUTE all platforms         →  cycle.status = COMPLETED, trigger CLEANUP_CYCLE
```

### Queue topology

- `audio.noise-remove` — dedicated, `prefetchCount=1` (ML GPU-heavy)
- `audio.pipeline.processing.high` — AUDIO_TRIM, AUDIO_CONCATENATE
- `audio.pipeline.processing.normal` — AUDIO_COMPRESS, AUDIO_DURATION, IMAGE_GENERATE, VIDEO_RENDER
- `audio.pipeline.processing.low` — DISTRIBUTE, CLEANUP_CYCLE

### Pipeline Run tracking

`pipeline_run` table (UUID PK) groups all `job_queue` rows for one invocation.
`job_queue.pipeline_run_id FK` enables per-run dashboards and retry isolation.

---

## Implementation plans

| Plan | File | Scope |
|---|---|---|
| A | `2026-05-08-pipeline-plan-a-audio-foundation.md` | `pipeline_run` table, JobType expansion, Python clients, audio runners, audio orchestration |
| B | `2026-05-08-pipeline-plan-b-image-video-distribution.md` | ImageGenerateJobRunner, VideoRenderJobRunner, DistributeJobRunner, full orchestrator |
| C | `2026-05-08-pipeline-plan-c-cleanup-e2e.md` | CleanupJobRunner, PipelineV1Controller, E2E integration test |
