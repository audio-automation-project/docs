# Local development runbook (golden path)

Minimal steps to run **PostgreSQL**, the **automation bot**, and the **admin frontend** on one machine. No Telegram, Firebase, qBittorrent, or GPU services required.

For variable reference, see [Configuration](configuration.md). Schema ↔ API mapping: [Schema sources of truth](../architecture/schema-sources-of-truth.md). **Separate Git repos per folder:** [Multi-repo daily workflow](../architecture/multi-repo-daily-workflow.md).

---

## Prerequisites

| Tool | Notes |
|------|--------|
| **Java 17** | `java -version` |
| **Maven** | Wrapper included: `./mvnw` (Unix) or `mvnw.cmd` (Windows) |
| **Node.js 18+** | For `audio-frontend` |
| **Docker** | Postgres on the golden path; optional Compose profile **`full-stack`** adds Redis + RabbitMQ (see Appendix B) |
| **FFmpeg** | Required when exercising **`/api/audio/*`** (compress, concatenate, …) |

---

## Golden path

### 1. Start PostgreSQL

From the **`audio-library-automation-bot`** directory:

```bash
docker compose up -d
```

Defaults (see `docker-compose.yml`): database **`audio_library`**, user **`audio`**, password **`audio`**, port **`5432`**.

### 2. Local Spring overrides (optional)

Copy **`env.example`** → **`application-local.properties`** in the same directory (or `config/application-local.properties`). Spring loads it via `spring.config.import` in `application.properties`.

For the golden path you often need **no extra keys** — JDBC defaults already match the compose file:

- URL: `jdbc:postgresql://localhost:5432/audio_library`
- User / password: **`audio`** / **`audio`**

Uncomment only what you need later (Telegram, OpenAI, etc.).

### 3. Run the backend

```bash
cd audio-library-automation-bot
./mvnw spring-boot:run
```

On Windows PowerShell:

```powershell
cd audio-library-automation-bot
.\mvnw.cmd spring-boot:run
```

Default HTTP port: **8088** (`server.port` / `SERVER_PORT` / Railway `PORT`).

**Core vs full stack**

| `app.core.enabled` | Effect |
|--------------------|--------|
| **Omitted** or **`true`** | “Core-only”: audio/library/job endpoints without loading Telegram, torrent, Playwright-heavy posters, etc. |
| **`false`** | Loads full integrations — requires real tokens and sibling services (Telegram, qBittorrent, …). |

Set in `application-local.properties`, for example:

```properties
# Golden path default — leave unset or:
app.core.enabled=true

# Full stack later:
# app.core.enabled=false
```

### 4. Run the frontend

```bash
cd audio-frontend
cp .env.example .env.local
npm install
npm run dev
```

Open **http://localhost:5173**. Ensure **`VITE_CORE_API_URL`** matches the bot (default `http://localhost:8088`). **`VITE_SILENCE_API_URL`** is optional unless you run the Demucs service (Appendix C).

---

## Verify

```bash
curl -s http://localhost:8088/actuator/health
curl -s http://localhost:8088/api/v1/library/audiobooks
curl -s "http://localhost:8088/api/v1/job-logs?limit=5"
```

Expect **`UP`** and JSON arrays (possibly empty) for the API calls.

In the UI: **Check Services**, **Library**, and **Jobs** tabs should load without browser Firebase access for those lists.

---

## Appendix A — Troubleshooting

| Issue | What to check |
|-------|----------------|
| **Cannot connect to Postgres** | `docker compose ps`; port **5432** not taken by another install; JDBC URL / credentials |
| **Flyway fails on startup** | Logs under Flyway; do not edit old migration files — add a new version if schema change is required |
| **Port 8088 in use** | `SERVER_PORT=8080 ./mvnw spring-boot:run` (Unix) or set env in Windows |
| **Frontend CORS** | Bot allows localhost Vite origins by default; set **`CORS_ALLOWED_ORIGINS`** for other hosts |
| **Windows paths** | Prefer forward slashes or escaped backslashes for **`AUDIO_BASE_PATH`** (see `env.example`) |

---

## Appendix B — Infra smoke (Postgres + Redis + RabbitMQ)

The automation bot always uses **PostgreSQL**. **Redis** and **RabbitMQ** are **optional**: when you activate Spring profile **`full-stack`** (matches `docker compose --profile full-stack`), the app enables Lettuce (Redis), declares **`audio.pipeline.jobs`**, publishes **`AUDIO_CONCATENATE`** jobs to that queue (instead of the in-process executor), and normally consumes those messages inside the JVM (**`PipelineJobsRabbitListener`**) unless **`APP_PIPELINE_RABBIT_CONSUMER_LOCAL_ENABLED=false`**. Optionally enable **`pipeline-worker`** (Python) plus **`APP_PIPELINE_WORKER_ENABLED`** — see subsection below — plus **actuator** health checks. With the default profile they stay off so you can run with Postgres only (`app.pipeline.rabbit-dispatch.enabled` is false).

### External `pipeline-worker` container

Build and run **`audio-library-automation-bot/pipeline-worker`** with Compose profile **`external-worker`** alongside **`full-stack`**:

```bash
docker compose --profile full-stack --profile external-worker up -d --wait
```

The worker reads JSON from **`audio.pipeline.jobs`** and calls **`POST /api/internal/v1/processing-jobs/{processingJobId}/run-concatenate`** on the bot with header **`X-Pipeline-Worker-Token`**. Configure the JVM with **`APP_PIPELINE_RABBIT_CONSUMER_LOCAL_ENABLED=false`** (otherwise the JVM listener and the worker will share the queue), **`SPRING_PROFILES_ACTIVE=full-stack`**, **`APP_PIPELINE_WORKER_ENABLED=true`**, and **`PIPELINE_WORKER_TOKEN`** equal to the worker’s env (**`dev-pipeline-worker-token`** matches Compose unless you override it). **`INTERNAL_BOT_URL`** in Compose defaults to **`http://host.docker.internal:8088`** for a bot listening on localhost.

Bring everything up:

```bash
cd audio-library-automation-bot
docker compose --profile full-stack up -d --wait
```

Run the scripted checks (PostgreSQL query, Redis `PING`, RabbitMQ HTTP API). From the **`audio-library-automation-bot`** directory:

```bash
./scripts/full-stack-smoke.sh
```

With HTTP checks against a bot you started separately (**`.\mvnw.cmd spring-boot:run`** on Windows), use profile **`full-stack`** so the JVM connects to Redis and RabbitMQ:

```bash
SPRING_PROFILES_ACTIVE=full-stack ./mvnw spring-boot:run
./scripts/full-stack-smoke.sh --bot-url http://localhost:8088
```

PowerShell: infra probes only:

```powershell
cd audio-library-automation-bot
.\scripts\full-stack-smoke.ps1
```

PowerShell: start the bot with **`full-stack`**, then HTTP smoke (two terminals):

```powershell
cd audio-library-automation-bot
$env:SPRING_PROFILES_ACTIVE = "full-stack"
.\mvnw.cmd spring-boot:run
```

```powershell
cd audio-library-automation-bot
.\scripts\full-stack-smoke.ps1 -BotBaseUrl http://localhost:8088
```

Equivalent manual curls (infra + bot) after services are running:

```bash
docker compose exec -T postgres psql -U audio -d audio_library -c "SELECT 1"
docker compose exec -T redis redis-cli ping
curl -sf -u audio:audio http://127.0.0.1:15672/api/overview | head -c 200 && echo ""
curl -s http://localhost:8088/actuator/health
curl -s http://localhost:8088/api/v1/library/audiobooks
curl -s "http://localhost:8088/api/v1/job-logs?limit=5"
```

Default RabbitMQ credentials in Compose: **`audio` / `audio`**. Ports: Postgres **5432**, Redis **6379**, Rabbit **5672** (AMQP) and **15672** (management UI).

---

## Appendix C — ML HTTP services (optional)

Use only when you need noise removal or Whisper transcription through the bot.

| Service | Folder | Default port | Notes |
|---------|--------|--------------|--------|
| **Demucs** | `audio_silence_service/` | **8000** | GPU typical; see service `README` / `docker compose` |
| **Whisper** | `whisper/` | **8002** | GPU typical |

Point the bot and frontend at running instances via env (see `env.example` and **`VITE_SILENCE_API_URL`**). These are **not** started by **`audio-library-automation-bot/docker-compose.yml`**.

---

## Appendix D — No Docker / Postgres elsewhere

The golden path assumes Docker runs local Postgres. If you use a cloud or host Postgres instead, set **`SPRING_DATASOURCE_URL`** (full JDBC URL with **`sslmode`** if required) and credentials per [Configuration](configuration.md).

There is no maintained **single-command H2 run profile** in `src/main/resources` for production-shaped Flyway + JPA today; use Postgres for parity with CI and Flyway.

---

## Related

- [Local development](local-development.md) — tests, builds, legacy notes
- [Configuration](configuration.md)
- [Frontend architecture](../architecture/frontend-architecture.md)
- [CLAUDE.md](../../CLAUDE.md) — workspace-level commands
