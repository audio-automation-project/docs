# Phase 3 — Text domain

**Plan reference:** [§5 Phase 3](../../2026-05-07-optimized-er-diagram-plan.md#phase-3--text-domain), [§2.3](../../2026-05-07-optimized-er-diagram-plan.md#23-text-domain)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** new migration(s) validated with Flyway on PostgreSQL; verification SQL in `psql`. No H2 requirement.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P3-T1 — Create `book_chapter`

| | |
|---|---|
| **Goal** | Add `book_chapter` with `cycle_id` FK, ordering constraints, and uniqueness `(cycle_id, chapter_number)`. |
| **Depends on** | P1-T2 (stable `cycles` identity) at minimum |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__book_chapter.sql` |

**Implementation notes**

- Columns per §2.3; FK `ON DELETE CASCADE` from plan.
- Add index for typical read pattern (`cycle_id`, `chapter_number`).

**Verification**

- Sample query from plan §2.3 “Full book text query” runs (empty result OK).
- `\d book_chapter` shows constraints.

**Tests**

- `./mvnw test`; add repository test if chapter persistence is introduced.

**Subagent brief**

> Add `book_chapter` table per plan §2.3. Target FK is to `cycles` until table rename to `cycle` occurs. Include indexes per §3.2 if listed for this table.
