# Design: Local dev runbook (golden path + optional ML)

**Date:** 2026-05-05  
**Status:** Accepted — implemented in `docs/guides/local-dev-runbook.md`  
**Scope:** Document how to bring up **Postgres + monolith + frontend** with minimal secrets; optional appendix for ML HTTP services.

## Goals

- Single ordered **golden path** that matches Flyway + JPA defaults in `application.properties`.
- Clarify **`app.core.enabled`**: heavy integrations load only when **`false`**; omit or **`true`** → core-only surface.
- Optional **Appendix B** for Demucs / Whisper — not part of default startup.

## Non-goals

- Adding new Docker services or changing Spring profiles in code (this iteration).
- Encoding cluster-split processes into default compose.

## Deliverables

| Artifact | Purpose |
|----------|---------|
| `docs/guides/local-dev-runbook.md` | Canonical steps, verification, troubleshooting, ML appendix |
| `audio-library-automation-bot/README.md` | Short Quick start + link (replace outdated Setup snippets) |
| `docs/guides/local-development.md` | Points to runbook; retains tests/build/commands where useful |

## Verification criteria

- New developer can follow runbook without Telegram, Firebase, qBittorrent, or GPU.
- `curl` checks document actual ports (**8088** default).
- ML paths reference **`audio_silence_service/`** and **`whisper/`** (folder names in repo).

## Self-review

- No TBD placeholders.
- Aligns with `docker-compose.yml` (`postgres:16-alpine`, user/db **audio** / **audio_library**).
- `DistributedToDto` / library APIs unchanged — runbook only.
