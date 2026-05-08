# Specs

Product and technical specifications for the Audiobook Automation Platform.

| Document | What it covers |
|---|---|
| [platform-product-plan.md](platform-product-plan.md) | Full product plan — problem, vision, current state, opportunities, phased roadmap, success metrics |
| [adr-001-javacv-over-ffmpeg-cli.md](adr-001-javacv-over-ffmpeg-cli.md) | Architecture decision: JavaCV replaces FFmpeg CLI — rationale, consequences, alternatives |
| [cycle-data-model.md](cycle-data-model.md) | Cycle entity spec — full JPA entities, enums, repositories, Flyway V1 migration SQL |

## Reading order for a new contributor

1. `platform-product-plan.md` — understand what we're building and why
2. `cycle-data-model.md` — understand the core data model before touching any service
3. `adr-001-javacv-over-ffmpeg-cli.md` — understand why media processing works the way it does
