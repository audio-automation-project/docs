# Optimized ER diagram — task index

**Parent plan:** [`2026-05-07-optimized-er-diagram-plan.md`](../../2026-05-07-optimized-er-diagram-plan.md)  
**Service:** `audio-library-automation-bot`  
**Flyway migrations directory:** `audio-library-automation-bot/src/main/resources/db/migration/`  
**Approved scope:** **PostgreSQL only** — see [Scope (PostgreSQL)](#scope-postgresql) below.

## Scope (PostgreSQL)

This task set was **database-first** for **Phases 1–9**: Flyway migrations and verification against **PostgreSQL** (same major version as production — not H2).

**Phase 10 (application alignment)** is **code-first** — see parent plan [§5 Phase 10](../../2026-05-07-optimized-er-diagram-plan.md). It intentionally changes Java, DTOs, optional frontend, and tests; migrations only if a schema fix is required.

| In scope (Phases 1–9) | Out of scope (historical) |
|----------|----------------------------------|
| New `V*.sql` migrations, enums, tables, FKs, triggers, procedures, `CALL` backfills | *(For phases 1–9 only:)* Java/JPA entities, repositories, services, REST DTOs |
| `psql` / pgAdmin / any PG client running the **Verification** SQL in each task | Spring `@Scheduled` callers *(phase 8 note: `sp_fail_stale_jobs` caller is now in app)* |
| Flyway `validate` / `migrate` against a local or CI PostgreSQL instance | Relying on `-Dspring-boot.run.profiles=dev` (H2) to prove PostgreSQL migrations |

| In scope (**Phase 10**) |
|----------|
| `LibraryAudiobookEntity` / `AudiobookDto` / Firestore mirror: `book_id`, `cycle_id`; `book_chapter` API; distribution write path; job FK population; frontend types/copy |

**Local PostgreSQL:** From `audio-library-automation-bot/`, use project `docker compose up -d` and the JDBC URL/credentials from that stack or `application-local.properties` (see [`CLAUDE.md`](../../../../../CLAUDE.md) / local runbook).

**Without Docker:** run `./mvnw test -Dtest=FlywayCycleSchemaIT` — migrations apply on **Zonky embedded PostgreSQL** (bundled Postgres binaries on the JVM, not Testcontainers).

**Definition of done (per task):** migration applies cleanly on PostgreSQL; verification queries pass; no edits to already-applied `V*.sql` files.

## Rules (from plan §5)

- Do not edit existing `V*.sql` files; add new migrations only with `VYYYYMMDDHHMM__name.sql`.
- Prefer nullable FKs first; tighten `NOT NULL` only after backfill verification.
- Each phase file lists **independent units** where possible; **dependencies** are explicit so subagents run in order within a phase.

## Phase files

| Order | File | Focus |
|------:|------|--------|
| 1 | [phase-01-bibliographic-core.md](phase-01-bibliographic-core.md) | `book`, `person`, join tables; link `cycles` + `library_audiobooks`; backfill |
| 2 | [phase-02-acquisition-normalization.md](phase-02-acquisition-normalization.md) | `source`, `book_source`; retire `scraped_books` |
| 3 | [phase-03-text-domain.md](phase-03-text-domain.md) | `book_chapter` |
| 4 | [phase-04-audio-stages.md](phase-04-audio-stages.md) | `audio_stage`; normalize `audio_parts` paths |
| 5 | [phase-05-video-stages.md](phase-05-video-stages.md) | `video_stage`; normalize `video_parts` paths |
| 6 | [phase-06-image-assets.md](phase-06-image-assets.md) | `image_asset`, `image_chunk`; `ai_content` shape |
| 7 | [phase-07-distribution.md](phase-07-distribution.md) | `platform`, `distribution`, `distribution_result` |
| 8 | [phase-08-job-queue.md](phase-08-job-queue.md) | **Done** — Flyway **V202605072455** (expand/rename to `job_queue`), **V202605072456** (`sp_fail_stale_jobs`), **V202605072457** (drop `job_logs`); app reads/writes `job_queue`, `/api/v1/job-logs` backed by `ProcessingJobEntity` |
| 9 | [phase-09-retire-legacy.md](phase-09-retire-legacy.md) | **Done** — **V202605072458** (`book_source.cycle_id` + backfill from legacy), **V202605072459** (drop `parsed_book`), **V202605072460** (drop backfill procedure + `_scraped_books_legacy`); parser/image-text/scraped-books API use `book` / `book_source` |
| 10 | *(see parent plan)* | **Application alignment** — parent plan [§5 Phase 10](../../2026-05-07-optimized-er-diagram-plan.md): map `library_audiobook.book_id` / `cycle_id` in JPA + DTOs + Firestore mirror; `book_chapter` API; distribution write path vs `distribution_result`; job_queue FK population; comment hygiene (`image_asset`); `audio-frontend` copy/types |

## Subagent-driven development

For each task block in a phase file, use this pattern:

1. Paste the **Subagent brief** into a new subagent with **only** that block (plus the parent plan link for DDL details).
2. **Implementer:** add the Flyway file(s); run **Flyway** against **PostgreSQL**; run each task’s **Verification** SQL in `psql` (or equivalent).
3. **Spec reviewer:** DDL and behavior match the parent plan sections cited in the task; objects exist in PostgreSQL `information_schema` / `pg_catalog` as expected.
4. **Code quality reviewer:** idempotent-safe migration style, naming per plan §6, no secrets in SQL, PostgreSQL-compatible syntax only.

Do not parallelize two tasks that list a **depends on** relationship.

**Optional:** `./mvnw test` only when the same change set updates Java; for **SQL-only PRs**, PostgreSQL + Flyway verification is sufficient.
