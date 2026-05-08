# Phase 5 — Video pipeline stages

**Plan reference:** [§5 Phase 5](../../2026-05-07-optimized-er-diagram-plan.md#phase-5--video-pipeline-stages), [§2.5 video / video_stage](../../2026-05-07-optimized-er-diagram-plan.md#25-production-domain)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** Flyway migrations targeting PostgreSQL; verify with `psql`. H2 not used to sign off this epic.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P5-T1 — Create `video_stage`

| | |
|---|---|
| **Goal** | Add `video_stage` table with `video_stage_type` (`PRE_RENDERED`, `FINAL`) and FK to `video_parts`. |
| **Depends on** | None |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__video_stage.sql` |

**Verification**

- Enum + table + unique `(video_part_id, stage)`.

**Tests**

- `./mvnw test`

**Subagent brief**

> Add `video_stage` per §2.5 referencing `video_parts(id)`.

---

## P5-T2 — Backfill `video_stage` from path columns

| | |
|---|---|
| **Goal** | Move `video_path` and `pre_rendered_segment_path` into stage rows. |
| **Depends on** | P5-T1 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__video_stage_backfill.sql` |

**Implementation notes**

- Map column → stage enum consistently in migration comments.

**Verification**

- Counts vs non-null legacy paths.

**Tests**

- `./mvnw test`

**Subagent brief**

> Backfill `video_stage` from `video_parts.video_path` and `pre_rendered_segment_path` per Phase 5 plan bullets.

---

## P5-T3 — Drop path columns from `video_parts`

| | |
|---|---|
| **Goal** | Remove `video_path`, `pre_rendered_segment_path` after backfill. |
| **Depends on** | P5-T2 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__video_parts_drop_paths.sql` |

**Verification**

- No video path columns on `video_parts`; code reads `video_stage`.

**Tests**

- `./mvnw test`; video generation smoke if applicable.

**Subagent brief**

> Drop legacy video path columns after backfill; update Java to use `video_stage` for file paths.
