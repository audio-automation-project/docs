# Audio Library System — Documentation

Central documentation for the Audio Library System — a microservices-based platform for audiobook production and distribution automation.

## Documentation Map

### Architecture
- [System Overview](architecture/system-overview.md) — concept, vision, service map, how services interact
- [Backend Architecture](architecture/backend-architecture.md) — Java packages, layered design, feature flags, design patterns
- [Frontend Architecture](architecture/frontend-architecture.md) — React dashboard, Firestore integration, pages & communication
- [ML Service Architecture](architecture/ml-service-architecture.md) — Python audio processing, Demucs, Whisper, SV2TTS voice cloning
- [Database Design](architecture/database-design.md) — entities, ER diagram, Firestore projection

### Pipeline
- [Audiobook Pipeline](pipeline/audiobook-pipeline.md) — 7-step production pipeline from discovery to publishing

### API
- [REST API Reference](api/rest-api-reference.md) — complete endpoint inventory with input/output specs

### Guides
- [Configuration](guides/configuration.md) — environment variables, feature flags, external dependencies
- [Local Development](guides/local-development.md) — how to set up and run each service
- [CI/CD](guides/ci-cd.md) — GitHub Actions workflows for validation and deployment

### Data
- [DTO Catalog](data/dto-catalog.md) — data transfer objects, mapping flows, boundary rules

## Platform at a Glance

| Service | Tech | Port | Role |
|---------|------|------|------|
| `audio-library-automation-bot` | Java 17, Spring Boot 3.4 | 8088 | Core backend — REST API, Telegram bot, FFmpeg, Playwright, AI |
| `audio-frontend` | React 19, TypeScript, Vite | 5173 | Dashboard — health checks, job monitoring, logs |
| `audio-silence-service` | Python 3.7+, PyTorch | 8090 | ML audio — voice separation, transcription, voice cloning |
