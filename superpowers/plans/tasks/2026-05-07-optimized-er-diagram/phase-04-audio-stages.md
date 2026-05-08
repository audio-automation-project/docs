# Phase 4 — Audio pipeline stages

**Plan reference:** [§5 Phase 4](../../2026-05-07-optimized-er-diagram-plan.md#phase-4--audio-pipeline-stages), [§2.5 audio / audio_stage](../../2026-05-07-optimized-er-diagram-plan.md#25-production-domain)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** DDL + backfill + drops validated on PostgreSQL via Flyway and **Verification** queries. Application code that reads path columns is **out of scope** here; open follow-ups as needed.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P4-T1 — Create `audio_stage` enum and table

| | |
|---|---|
| **Goal** | Introduce `audio_stage` with `audio_stage_type` enum and uniqueness `(audio_part_id, stage)`. |
| **Depends on** | None (relies on existing `audio_parts`) |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__audio_stage.sql` |

**Implementation notes**

- Map plan enum names to DB; avoid clashing with existing `CREATE TYPE` names.
- FK `audio_part_id` → `audio_parts(id)` `ON DELETE CASCADE`.

**Verification**

- Table and enum exist; test insert one row per stage type allowed.

**Tests**

- `./mvnw test`

**Subagent brief**

> Create `audio_stage` per plan §2.5. Reference current plural table name `audio_parts`. Add constraints from §3.1 if any for file_path.

---

## P4-T2 — Backfill `audio_stage` from legacy path columns

| | |
|---|---|
| **Goal** | Insert one row per non-null path in `audio_parts` for `ORIGINAL`, `CONCATENATED`, `TRIMMED`, `COMPRESSED` (and any others defined in migration). |
| **Depends on** | P4-T1 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__audio_stage_backfill_from_columns.sql` |

**Implementation notes**

- Source columns today: `original_chapters_path`, `concatenated_audio_path`, `trimmed_audio_path`, `compressed_audio_path` (per `V202605052130__...`).
- Use `INSERT ... SELECT WHERE path IS NOT NULL`.

**Verification**

- Row count sanity: sum of non-empty path columns ≈ rows inserted (account for duplicates).
- Spot-check N random `audio_parts` IDs.

**Tests**

- Flyway + `./mvnw test`

**Subagent brief**

> Backfill `audio_stage` from legacy `audio_parts` path columns. Only insert where path non-null. Document mapping from column → `audio_stage_type` in migration header.

---

## P4-T3 — Drop legacy path columns from `audio_parts`

| | |
|---|---|
| **Goal** | Remove the four path columns after backfill verified. |
| **Depends on** | P4-T2 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__audio_parts_drop_path_columns.sql` |

**Implementation notes**

- Grep Java/Python for column mappings before merge; update entities and services in same or prerequisite PR.

**Verification**

- `\d audio_parts` shows no path columns.
- Application can resolve paths via `audio_stage` only.

**Tests**

- `./mvnw test` full suite; manual audio pipeline smoke if available.

**Subagent brief**

> Drop original/concatenated/trimmed/compressed path columns from `audio_parts` per plan Phase 4 after backfill. Ensure application code no longer references dropped columns.
