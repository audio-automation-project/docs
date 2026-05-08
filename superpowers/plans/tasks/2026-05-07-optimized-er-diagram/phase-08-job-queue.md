# Phase 8 — Job queue expansion

**Plan reference:** [§5 Phase 8](../../2026-05-07-optimized-er-diagram-plan.md#phase-8--job-queue-expansion), [§2.7](../../2026-05-07-optimized-er-diagram-plan.md#27-operations-domain), [§3.4 sp_fail_stale_jobs](../../2026-05-07-optimized-er-diagram-plan.md#sp_fail_stale_jobsp_older_than_minutes-int--reaps-stuck-in_progress-jobs)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** `job_queue` DDL, `sp_fail_stale_jobs`, drop `job_logs` — Flyway + PostgreSQL. Spring `@Scheduled` caller for `CALL sp_fail_stale_jobs` is **out of scope** (separate ticket). H2 not used for sign-off.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P8-T1 — Expand `processing_jobs` schema toward `job_queue`

| | |
|---|---|
| **Goal** | Add nullable FK columns `video_part_id`, `image_asset_id`, `ai_content_id`, `distribution_id`; add `started_at`, `completed_at`, `error_message`; align types with §2.7; rename table to `job_queue`. |
| **Depends on** | P5-T1 (video_part), P6-T1 (image_asset), P6-T5 (ai_content), P7-T2 (distribution) — schedule after those tables exist |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__job_queue_expand_and_rename.sql` |

**Implementation notes**

- Existing table from earlier migrations may use different PK type — plan says UUID PK for `job_queue`; if current PK is UUID already, match; if BIGINT, document deviation or add migration to alter PK **only** if approved (may be out of scope; prefer additive columns first).
- FKs `ON DELETE SET NULL` per §2.7.

**Verification**

- `\d job_queue` matches intended columns; old name `processing_jobs` gone or synonym not used.

**Tests**

- `./mvnw test`; update entities and job services.

**Subagent brief**

> Migrate `processing_jobs` toward `job_queue` per plan §5 Phase 8 and §2.7. Add typed FKs and timestamps. Rename table. Do not break existing job rows: new columns nullable.

---

## P8-T2 — Add `sp_fail_stale_jobs` (and optional Spring schedule hook)

| | |
|---|---|
| **Goal** | Provide procedure §3.4 for stale `IN_PROGRESS` jobs; optional separate Java task to call `CALL sp_fail_stale_jobs(60)` per open question §7.6. |
| **Depends on** | P8-T1 |
| **Where to implement** | SQL: `V...__sp_fail_stale_jobs.sql`; Java (if in scope): `kg.automation...` scheduled component — **only if user approves app change in same epic** |

**Verification**

- Manual: insert fake stale row, call procedure, status becomes `FAILED` with message.

**Tests**

- If Java added: unit test with mocked JDBC or integration test; else SQL-only verification documented.

**Subagent brief**

> Create `sp_fail_stale_jobs` per §3.4. Document optional Spring `@Scheduled` wiring per plan §7.6 in commit message or small Java PR.

---

## P8-T3 — Drop `job_logs`

| | |
|---|---|
| **Goal** | Remove `job_logs`; operational history lives on `job_queue` per plan §1. |
| **Depends on** | P8-T1 (data model can represent status history — confirm product accepts loss of historical rows or archive first) |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__drop_job_logs.sql` |

**Implementation notes**

- If retention required: export/archive migration step **before** DROP in prior task.

**Verification**

- Table absent; no code references.

**Tests**

- `./mvnw test`

**Subagent brief**

> Drop `job_logs` per Phase 8 after archival decision. Remove Java repositories using `job_logs`.
