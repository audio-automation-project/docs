# Phase 7 — Distribution split + platform table

**Plan reference:** [§5 Phase 7](../../2026-05-07-optimized-er-diagram-plan.md#phase-7--distribution-split--platform-table), [§2.6](../../2026-05-07-optimized-er-diagram-plan.md#26-distribution-domain), [§3.3 fn_sync_distribution_status](../../2026-05-07-optimized-er-diagram-plan.md#fn_sync_distribution_status--auto-updates-distributionstatus-from-results)

**Repository root:** `audio-library-automation-bot/`

## Scope (PostgreSQL — approved)

**PostgreSQL only:** platform seed, distribution DDL, triggers/functions, data migration from `distribution_records` — all on PostgreSQL via Flyway; verify with `psql`. Java uploaders unchanged in this scope.

**Local DB:** `docker compose up -d` from `audio-library-automation-bot/`.

---

## P7-T1 — Create `platform` and seed rows

| | |
|---|---|
| **Goal** | Add `platform` table; insert five seeded platforms with `TWITCH`/`PATREON` `enabled = false` per §2.6. |
| **Depends on** | None |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__platform.sql` |

**Verification**

- `SELECT code, enabled FROM platform ORDER BY id` matches seed table in plan.

**Tests**

- `./mvnw test`

**Subagent brief**

> Create `platform` per §2.6 with JSONB `config` default. Seed IDs 1–5 as documented; disabled flags for Twitch/Patreon.

---

## P7-T2 — Create `distribution` and `distribution_result`; migrate from `distribution_records`

| | |
|---|---|
| **Goal** | New tables per §2.6; data migration from legacy `distribution_records`; partial unique indexes for nullable `audio_part_id`; create `fn_sync_distribution_status` + trigger on `distribution_result`. |
| **Depends on** | P7-T1; existing `cycles` / `audio_parts` for FK validity |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__distribution_split.sql` (split into multiple migrations if size warrants) |

**Implementation notes**

- Include `sp_refresh_distribution_status` from §3.4 if not already present (optional same file).
- Map old enum `distribution_platform` / `distribution_status` to new `release_status` / `platform_result_status`.

**Verification**

- Row counts: migrated distributions ≈ source table (document transformations).
- Trigger test: insert results → parent `distribution.status` updates per rules.

**Tests**

- `./mvnw test`; repository tests for distribution persistence.

**Subagent brief**

> Implement distribution split per plan §5 Phase 7 and DDL §2.6. Add status sync function/trigger §3.3. Migrate `distribution_records` with explicit column mapping in migration comments.

---

## P7-T3 — Drop Telegram-specific tables

| | |
|---|---|
| **Goal** | Remove `telegram_media_groups`, `telegram_media_items`; data folded into `distribution_result.platform_metadata` per plan §1. |
| **Depends on** | P7-T2 (telemetry paths updated) |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__drop_telegram_media_tables.sql` |

**Verification**

- Tables dropped; Java no longer references them.

**Tests**

- `./mvnw test`

**Subagent brief**

> Drop telegram tables per Phase 7 after confirming distribution_results capture needed metadata.

---

## P7-T4 — Drop `distribution_records`

| | |
|---|---|
| **Goal** | Remove legacy table after successful migration and code cutover. |
| **Depends on** | P7-T2, P7-T3 optional |
| **Where to implement** | `src/main/resources/db/migration/VYYYYMMDDHHMM__drop_distribution_records.sql` |

**Verification**

- Table absent.

**Tests**

- `./mvnw test`

**Subagent brief**

> Drop `distribution_records` only when application reads/writes `distribution` / `distribution_result` only.
