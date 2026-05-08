# Phase 9 — Retire legacy tables

**Plan reference:** [§5 Phase 9](../../2026-05-07-optimized-er-diagram-plan.md#phase-9--retire-legacy-tables), [§1 Current tables](../../2026-05-07-optimized-er-diagram-plan.md#1-current-tables-reference-only)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** legacy table drops and checklist after PostgreSQL verification. Ensure no remaining FKs from other schemas before `DROP`. Grep application code separately — **out of scope** for DB-only subagents.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P9-T1 — Drop `parsed_book`

| | |
|---|---|
| **Goal** | Remove legacy acquisition table after any backfill to `book` / `book_source` is complete and unused by code. |
| **Depends on** | P1–P2 stable; grep codebase for `parsed_book` = 0 hits |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__drop_parsed_book.sql` |

**Verification**

- `to_regclass('public.parsed_book')` is null.

**Tests**

- `./mvnw test`

**Subagent brief**

> Drop `parsed_book` per plan §1 only when no code or FK references remain. Confirm with repository-wide search.

---

## P9-T2 — Drop `_scraped_books_legacy`

| | |
|---|---|
| **Goal** | Remove renamed legacy table after retention policy satisfied. |
| **Depends on** | P2-T3 was rename; P2-T2 backfill done; code never reads legacy |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__drop_scraped_books_legacy.sql` |

**Verification**

- Legacy table absent.

**Tests**

- `./mvnw test`

**Subagent brief**

> Drop `_scraped_books_legacy` per Phase 9. Ensure `book_source` has needed historical fields before drop.

---

## P9-T3 — Post-schema checklist (documentation only in PR description)

| | |
|---|---|
| **Goal** | Confirm optional follow-ups from plan §7 open questions (library_audiobook denormalized strings, platform_metadata validation, etc.) are either resolved or ticketed. |
| **Depends on** | P9-T1, P9-T2 |
| **Where to implement** | No migration; **PR checklist** or issue tracker |

**Verification**

- Product/sign-off recorded.

**Tests**

- N/A

**Subagent brief**

> Review `## 7. Open questions` in parent plan; close or create issues. No DDL.
