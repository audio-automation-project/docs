# Phase 6 — Unified image storage

**Plan reference:** [§5 Phase 6](../../2026-05-07-optimized-er-diagram-plan.md#phase-6--unified-image-storage), [§2.5 image_asset / image_chunk / ai_content](../../2026-05-07-optimized-er-diagram-plan.md#25-production-domain), triggers [§3.3](../../2026-05-07-optimized-er-diagram-plan.md#33-functions-and-triggers)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** tables, triggers, backfills, drops — Flyway + PostgreSQL + task verification SQL. JPA entity renames **out of scope** for the same subagent unless explicitly split.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P6-T1 — Create `image_asset` and `image_chunk` (+ chunk sync trigger)

| | |
|---|---|
| **Goal** | Add `image_asset`, `image_chunk`, CHECK rules from §3.1, indexes, and `fn_validate_image_asset_chunks` + trigger on `image_chunk`. |
| **Depends on** | None |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__image_asset_and_chunk.sql` |

**Implementation notes**

- Reconcile enum names with existing `image_part_type` / storage model.
- Implement trigger per §3.3 `fn_validate_image_asset_chunks`.

**Verification**

- Insert parent + chunks; `chunk_count` updates after insert/delete.

**Tests**

- `./mvnw test`

**Subagent brief**

> Create `image_asset` and `image_chunk` per §2.5 and constraints §3.1. Add chunk-count sync trigger §3.3. FK to `cycles` until rename.

---

## P6-T2 — Backfill from `image_parts` to `image_asset`

| | |
|---|---|
| **Goal** | Copy existing image rows into `image_asset` (map `image_type` → `asset_type`, paths/base64/chunks). |
| **Depends on** | P6-T1 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__image_asset_backfill_from_image_parts.sql` |

**Verification**

- `COUNT(*)` `image_asset` ≥ `image_parts`; spot-check MIME and `storage_type`.

**Tests**

- Flyway + `./mvnw test`

**Subagent brief**

> INSERT...SELECT from `image_parts` into `image_asset` with explicit enum mapping. Handle `CHUNKED_BASE64` vs `INLINE` per legacy columns.

---

## P6-T3 — Backfill `image_chunk` from `image_base64_chunks`

| | |
|---|---|
| **Goal** | Move chunk rows to reference new `image_asset_id`. |
| **Depends on** | P6-T2 (stable ID mapping — store legacy mapping in temp column or join via deterministic order) |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__image_chunk_backfill.sql` |

**Implementation notes**

- Plan §1: old `image_base64_chunks` had polymorphic parent — migration must join via old FKs to new `image_asset.id`.

**Verification**

- Chunk totals per asset match legacy counts.

**Tests**

- `./mvnw test`

**Subagent brief**

> Backfill `image_chunk` from `image_base64_chunks` joining through backfilled `image_asset` rows. Verify `chunk_index` uniqueness.

---

## P6-T4 — Backfill IMAGE-type rows from `ai_generated_content` into `image_asset`

| | |
|---|---|
| **Goal** | Relocate AI image artifacts from `ai_generated_content` to `image_asset`; leave text rows for next task. |
| **Depends on** | P6-T1 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__image_asset_backfill_from_ai_content.sql` |

**Verification**

- No remaining `content_type = 'IMAGE'` rows needing file data **or** intentional exceptions documented.

**Tests**

- `./mvnw test`

**Subagent brief**

> Migrate IMAGE rows from `ai_generated_content` to `image_asset` per plan §1 fate table and §2.5. Preserve `cycle_id` linkage.

---

## P6-T5 — Narrow `ai_generated_content` to text types; rename to `ai_content`

| | |
|---|---|
| **Goal** | Remove IMAGE from enum; drop obsoleted storage columns if any; rename table to `ai_content`. |
| **Depends on** | P6-T4 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__ai_content_narrow_and_rename.sql` |

**Implementation notes**

- Use PostgreSQL-safe enum alteration pattern (`ALTER TYPE ...`) or replace type per DBA standard.
- Coordinate Java entity rename.

**Verification**

- Only `TEXT` / `DESCRIPTION` allowed; table name `ai_content`.

**Tests**

- `./mvnw test`

**Subagent brief**

> Apply `ai_content_drop_image.sql` intent from Phase 6 plan: enum without IMAGE, column cleanup, table rename. Update JPA `@Table` names.

---

## P6-T6 — Drop legacy image tables

| | |
|---|---|
| **Goal** | Drop `image_parts` and `image_base64_chunks` after cutover. |
| **Depends on** | P6-T2, P6-T3, application code switched |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__drop_image_parts_and_chunks.sql` |

**Verification**

- Tables absent; no FK references.

**Tests**

- `./mvnw test`

**Subagent brief**

> Drop `image_parts` and `image_base64_chunks` per plan §5 Phase 6 after backfill + code migration.
