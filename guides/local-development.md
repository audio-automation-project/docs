# Local Development

Use **[Local dev runbook](local-dev-runbook.md)** for the **golden path** (Docker Postgres → `./mvnw spring-boot:run` → `npm run dev`) and verification curls.

This page keeps **build/test commands** and **extended ML** notes.

## Prerequisites

| Tool | Version | Required For |
|------|---------|-------------|
| Java JDK | 17+ | Backend |
| Maven | 3.8+ | Backend (or use `mvnw` in `audio-library-automation-bot`) |
| Node.js | 18+ | Frontend |
| npm | 9+ | Frontend dependencies |
| Docker | Current | Postgres on golden path |
| FFmpeg | Latest | `/api/audio/*` when exercised |
| Python | 3.9+ | Optional ML services |

## Backend — tests and builds

From repository root or `audio-library-automation-bot`:

```bash
./mvnw test
./mvnw clean package -DskipTests
```

Windows: `.\mvnw.cmd test`

Full stack (`app.core.enabled=false`) needs real secrets — see [Configuration](configuration.md).

## Frontend (`audio-frontend`)

```bash
cd audio-frontend
npm install
npm run dev
```

Opens at **http://localhost:5173**. Copy **`.env.example`** → **`.env.local`** (`VITE_CORE_API_URL`, optional **`VITE_SILENCE_API_URL`**).

```bash
npm run build
npm run lint
npm test
```

## ML services (optional)

HTTP microservices live under **`audio_silence_service/`** (Demucs, port **8000**) and **`whisper/`** (port **8002**). See each folder’s `README` / `docker compose`. The bot compose file does **not** start them — details in [Local dev runbook — Appendix B](local-dev-runbook.md#appendix-b--ml-http-services-optional).

Legacy/offline scripts may exist under other paths; prefer the FastAPI services for integration with the bot.

## Verifying

1. **Backend**: `curl http://localhost:8088/actuator/health`
2. **Frontend**: dashboard **Check Services** / Library / Jobs

## Related Documentation

- [Local dev runbook](local-dev-runbook.md) — start here
- [Configuration](configuration.md)
- [CI/CD](ci-cd.md)
- [ML Service Architecture](../architecture/ml-service-architecture.md)
