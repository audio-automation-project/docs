# Phase 1 — Bibliographic core

**Plan reference:** [§5 Phase 1](../../2026-05-07-optimized-er-diagram-plan.md#phase-1--bibliographic-core), schema [§2.2](../../2026-05-07-optimized-er-diagram-plan.md#22-bibliographic-domain), procedure [§3.4](../../2026-05-07-optimized-er-diagram-plan.md#34-stored-procedures)

**Repository root (relative paths):** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** author and validate all migrations with Flyway **against PostgreSQL** (not H2). Use `psql` (or any PostgreSQL client) for task **Verification** queries. Java/JPA changes are **out of scope** unless you open a separate epic; note follow-ups in migration comments if drops/renames will break the app until wired.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`; use that JDBC URL for Flyway.

---

## P1-T1 — Create bibliographic tables and enums

| | |
|---|---|
| **Goal** | Introduce `book`, `person`, `book_person`, `genre`, `book_genre` with indexes and `updated_at` triggers per plan §2.2 and §3.3. |
| **Depends on** | None |
| **Where to implement** | New Flyway: `src/main/resources/db/migration/VYYYYMMDDHHMM__book_bibliographic_core.sql` (pick next timestamp after latest file). |
| **Specification** | Plan §2.2 (column types, PKs, FK rules), §3.1 CHECKs if any for these tables, §6 naming. Reconcile new enum types with existing DB enums (e.g. do not duplicate `CREATE TYPE` names already present from `V202605052130__...`). |

**Implementation notes**

- Include `CREATE TYPE` only for closed sets defined in the plan (e.g. `book_person_role`) if not already present.
- Use `GENERATED ALWAYS AS IDENTITY` and `public_id` defaults as in plan.
- Attach `trg_*_updated_at` to every table that has `updated_at` per §3.3.

**Verification**

- `SELECT tablename FROM pg_tables WHERE schemaname = 'public' AND tablename IN ('book','person','book_person','genre','book_genre');` — all present.
- Flyway: `./mvnw -pl . flyway:info` (or full project flyway goal as configured) — migration applied once, no error.
- Constraint smoke: insert one `book`, one `person`, one `book_person` row in a transaction and roll back.

**Tests**

- If JPA entities are added in the same PR: unit/repository tests; otherwise `./mvnw test` on unchanged suite must still pass.

**Subagent brief**

> Add Flyway migration `V...__book_bibliographic_core.sql` creating bibliographic tables per `docs/superpowers/plans/2026-05-07-optimized-er-diagram-plan.md` §2.2. Reuse existing `set_updated_at()` if it exists; do not recreate it under a conflicting signature. Add triggers and indexes per §3.3 / §3.2. Do not modify earlier migration files.

---

## P1-T2 — Nullable `book_id` on `cycles`

| | |
|---|---|
| **Goal** | Add nullable `book_id BIGINT` referencing `book(id)` with appropriate `ON DELETE` per target ER (plan §2.5). |
| **Depends on** | P1-T1 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__cycles_add_book_id.sql` |

**Implementation notes**

- Current table name in DB is `cycles` (plural); keep until a later rename migration if the plan requires singular `cycle`.
- Add index per plan §3.2 patterns (`ix_cycles_book` or equivalent).

**Verification**

- `\d cycles` (or `information_schema`) shows nullable `book_id` FK.
- Optional: `SELECT COUNT(*) FROM cycles WHERE book_id IS NOT NULL` = 0 after migration (before backfill).

**Tests**

- `./mvnw test` — no regression.

**Subagent brief**

> Add Flyway migration adding nullable `book_id` to existing `cycles` table per optimized ER plan §2.5 and Phase 1 bullet `cycle_book_fk.sql`. Use FK to `book(id)`. Add supporting index. Do not set `NOT NULL` yet.

---

## P1-T3 — Nullable `book_id` and `cycle_id` on `library_audiobooks`

| | |
|---|---|
| **Goal** | Add nullable FKs `book_id` → `book`, `cycle_id` → `cycles` per plan §2.8 catalog bridge. |
| **Depends on** | P1-T1, P1-T2 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__library_audiobooks_add_book_cycle_fk.sql` |

**Implementation notes**

- Table may be `library_audiobooks` today; match actual name from existing Flyway.
- Partial indexes (`WHERE ... IS NOT NULL`) as in plan §3.2 if specified.

**Verification**

- FK constraints exist; columns nullable.
- No orphaned requirement until backfill — counts may be zero links.

**Tests**

- `./mvnw test`

**Subagent brief**

> Add migration for nullable `book_id` and `cycle_id` on operational catalog table per plan §2.8. Reference actual table name used in `V202605052210__library_audiobooks.sql`. Add indexes per plan §3.2.

---

## P1-T4 — Procedure and execution: `sp_backfill_book_from_cycles`

| | |
|---|---|
| **Goal** | Create `sp_backfill_book_from_cycles()` and run it once in controlled fashion so every `cycles` row with data gets a `book_id`. |
| **Depends on** | P1-T1, P1-T2 |
| **Where to implement** | Prefer two files if safer: (a) `V...__sp_backfill_book_from_cycles.sql` — `CREATE OR REPLACE PROCEDURE ...`; (b) `V...__call_backfill_book_from_cycles.sql` — `CALL sp_backfill_book_from_cycles();` OR single file with both — team standard. |

**Implementation notes**

- SQL body in plan §3.4 must be aligned with real constraints on `book` (e.g. `ON CONFLICT` targets must match actual UNIQUE indexes on `book` / `person`).
- Document in migration comment if manual step is required in prod.

**Verification**

- After `CALL`: `SELECT COUNT(*) FROM cycles WHERE book_id IS NULL AND title IS NOT NULL` should be **0** (or document exceptions).
- `SELECT COUNT(*) FROM book >= 1` if any titled cycles existed.

**Tests**

- Integration test with testcontainers/H2 may not run procedures; at minimum Flyway succeeds on PostgreSQL dev DB.

**Subagent brief**

> Implement `sp_backfill_book_from_cycles` per plan §3.4. Fix procedure logic if plan’s `ON CONFLICT` does not match §2.2 constraints (add missing UNIQUE or adjust INSERT). Add Flyway migration(s) to create procedure and invoke `CALL` once. Confirm all cycles with title are linked.

---

## P1-T5 — Tighten `cycles.book_id` and drop denormalized columns

| | |
|---|---|
| **Goal** | Set `cycles.book_id` `NOT NULL` when safe; drop `cycles.title` and `cycles.author` per Phase 1 closing bullet. |
| **Depends on** | P1-T4 (backfill complete); P1-T3 can remain parallel if only cycles change |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__cycles_book_id_not_null_drop_title_author.sql` |

**Implementation notes**

- **Reconciliation:** Target §2.5 still shows `title` on `cycle`. Before dropping `title`, confirm product intent (denormalized part title vs book title). If retained, migrate column semantics instead of drop — escalate to plan author if plan contradicts §2.5.
- Handle orphan rows: either synthetic book or block migration with clear error.

**Verification**

- `book_id` `NOT NULL` on `cycles`.
- Columns `title`/`author` absent **or** decision documented in migration preamble.

**Tests**

- `./mvnw test`; manual API smoke if Java still maps dropped columns.

**Subagent brief**

> After successful backfill, add migration to set `cycles.book_id NOT NULL` and drop `title`/`author` per plan §5 Phase 1 — only if product sign-off matches §2.5. Include pre-check `DO $$ ... $$` raising exception if NULL `book_id` remains.

---

## P1-T6 — Backfill `library_audiobooks` keys from existing strings (optional bridge)

| | |
|---|---|
| **Goal** | Populate `library_audiobooks.book_id` / `cycle_id` where deterministically matchable (e.g. by `cycle_identifier` or title) so API can migrate off string cache. |
| **Depends on** | P1-T3, P1-T4 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__library_audiobooks_backfill_fks.sql` (procedure or `UPDATE ... FROM`). |

**Implementation notes**

- Plan §7 open question #1: strings remain until stable — this task is optional and may be a no-op migration with comment if no reliable key.

**Verification**

- Row counts: linked vs unlinked catalog rows documented in migration comment or log table.

**Tests**

- `./mvnw test`

**Subagent brief**

> If matching rules are defined by the team, add optional backfill migration for `library_audiobooks` FKs using joins through `cycles`/`book`. Otherwise add commented no-op and reference open question §7.1 in parent plan.
