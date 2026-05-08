# Phase 2 — Acquisition normalization

**Plan reference:** [§5 Phase 2](../../2026-05-07-optimized-er-diagram-plan.md#phase-2--acquisition-normalization), [§2.4](../../2026-05-07-optimized-er-diagram-plan.md#24-acquisition-domain)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** Flyway + `psql` verification; do not use H2 as proof these migrations work. Java persistence updates are **out of scope** for these tasks — if a rename breaks ORM, record a follow-up ticket.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P2-T1 — Create `source` and `book_source`

| | |
|---|---|
| **Goal** | Add `source` and `book_source` with constraints per §2.4 (`source_type` CHECK, JSONB default, `legacy_scraped_book_id` partial unique if specified). |
| **Depends on** | P1-T4 recommended (needs `book_id` on cycles for later backfill join) — can run after P1-T1 if only DDL this task |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__source_and_book_source.sql` |

**Implementation notes**

- Seed minimal `source` row only if Phase 2 procedure migration expects it; otherwise seed in P2-T2.
- Match FK `book_id` → `book`, `source_id` → `source`.

**Verification**

- Tables exist; FKs valid; `INSERT` smoke for one source + one book_source in transaction rollback.

**Tests**

- `./mvnw test`

**Subagent brief**

> Create Flyway migration for `source` and `book_source` per plan §2.4. Use naming §6. Do not alter `scraped_books` in this file.

---

## P2-T2 — Procedure `sp_backfill_book_source_from_scraped_books` + CALL

| | |
|---|---|
| **Goal** | Migrate rows from `scraped_books` into `book_source`, linking via `cycles.book_id`. |
| **Depends on** | P2-T1, P1-T4 |
| **Where to implement** | `V...__sp_backfill_book_source.sql` + optional `V...__call_backfill_book_source.sql` |

**Implementation notes**

- Procedure SQL in §3.4 joins `scraped_books` to `cycles`; requires `cycles.book_id` populated.
- Ensure `source` row for `baza-knig` exists before `CALL`.

**Verification**

- `SELECT COUNT(*) FROM book_source` >= rows in `scraped_books` with matching `cycle_id` and non-null `book_id`.
- `legacy_scraped_book_id` unique when not null.

**Tests**

- Flyway on dev PostgreSQL; `./mvnw test`

**Subagent brief**

> Implement procedure from plan §3.4 and invoke once via Flyway. Align conflict targets with actual unique indexes on `book_source`. Verify counts vs `scraped_books`.

---

## P2-T3 — Retire `scraped_books` (rename to legacy)

| | |
|---|---|
| **Goal** | Stop live use of old table by renaming to `_scraped_books_legacy` per Phase 2 bullet. |
| **Depends on** | P2-T2 |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__scraped_books_rename_legacy.sql` |

**Implementation notes**

- Update Java `@Table` / native queries / repositories that reference `scraped_books` **in the same PR or a prior PR** — list as blocker in task if separate.

**Verification**

- `SELECT to_regclass('public.scraped_books')` is null; legacy name exists.

**Tests**

- `./mvnw test` must pass with code updated to new table name or abstraction.

**Subagent brief**

> Rename `scraped_books` to `_scraped_books_legacy` per plan §5 Phase 2. Coordinate with Java persistence layer in same change set so build stays green.
