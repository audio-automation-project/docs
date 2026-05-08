# Audiobook Automation Platform — Product Plan
### Status: Active Development · Version 1.0 · 2026-05-05

---

## 1. Why This Exists (The Problem)

Producing and distributing audiobooks manually is a multi-hour, multi-tool grind:

- Find a source → download via torrent → join audio chapters → clean noise → generate cover art → write description → encode video → post to Boosty → upload to YouTube → send to Telegram

Every one of those steps today is a separate manual action, across separate tools, with no shared state. Drop one step, start over. The platform automates the entire chain — one trigger, one result.

**The real pain is not any single step. It is the coordination between steps.** State gets lost between tools. Files end up in the wrong place. The same metadata gets typed multiple times. A failed upload at step 7 means manually picking up from there.

**Root cause:** there is no single system of record for a "job". No Cycle entity that owns the state from search to publish.

---

## 2. Who This Is For

**Primary user:** The operator running this platform — someone who manages audiobook content production and wants to go from "I want to publish this book" to "it's live on Boosty, YouTube, and Telegram" with minimal intervention.

**Secondary user:** Future operators or contributors who need to understand, extend, or debug the pipeline.

The platform is not consumer-facing. There is no sign-up flow, no public API. It is an internal automation system.

---

## 3. What Success Looks Like

**In 3 months:**
- A single REST call (or Telegram command) starts a full cycle for any book
- The cycle's state is visible at every stage via the admin frontend
- Failures are caught, logged, and resumable — not silent dead ends
- Deployment to Railway works with `git push` — no manual server config

**In 6 months:**
- All 11 microservices running independently, wired through the API Gateway
- Zero dependency on installed CLI tools (FFmpeg binary, ffprobe) — all media processing via JavaCV bundled in Maven
- Boosty, YouTube, and Telegram posts go live automatically when a cycle completes

**In 12 months:**
- Multiple concurrent cycles run without interfering
- Full observability: logs, job status, error replay via the admin dashboard
- New distribution platforms can be added without touching existing services

---

## 4. Current State (What Exists Today)

### What works and is production-ready
| Component | Status | Notes |
|---|---|---|
| `audio-library-automation-bot` | ✅ Running | Spring Boot 3.4 monolith, all logic inside |
| `audio_silence_service` | ✅ Running | Python + Demucs, GPU noise removal, port 8000 |
| `whisper` | ✅ Running | Python + Whisper, transcription, port 8002 |
| `audio-frontend` | ✅ Running | React 19 admin dashboard, Firebase real-time |
| PostgreSQL | ✅ Connected | Via Docker Compose, Flyway partially enabled |

### What is in the monolith and needs extraction
- Audio processing (FFmpeg via `ProcessBuilder` — CLI dependency)
- Image processing (Java AWT — already self-contained)
- AI clients (OpenAI + Ollama — REST calls)
- Telegram bot (TelegramLongPollingBot)
- Video creation (FFmpeg 2-step — CLI dependency)
- Boosty posting (Playwright browser automation)
- YouTube upload (Google API client)
- Web scraping (Playwright + Jsoup)
- Torrent management (qBittorrent Web API)

### Known problems in the current monolith
1. **`AudioConcatenatorService` uses a hardcoded `filePath.txt`** — concurrent jobs corrupt each other's file lists
2. **JSON file storage** — no transactions, no query capability, read entire file on every write
3. **FFmpeg via `ProcessBuilder`** — requires FFmpeg binary installed in the container; breaks portability
4. **`~100MB Node.js` binaries committed in Java source** — inside `poster/authorization/Service/node/`
5. **No Cycle entity** — job state is scattered across JSON files and in-memory variables
6. **FFmpeg runs on the API thread** — long jobs block HTTP responses

---

## 5. The Vision — Multi-Module Microservices Platform

### Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
                        │          api-gateway-service :8088           │
                        │          (Spring Cloud Gateway)              │
                        └──────────────────┬──────────────────────────┘
                                           │ routes all /api/* traffic
          ┌────────────────────────────────┼─────────────────────────────────┐
          │                               │                                   │
   ┌──────▼──────┐              ┌─────────▼────────┐             ┌───────────▼────────┐
   │core-service │              │torrent-downloader│             │ content-search     │
   │   :8080     │              │    :8081         │             │    :8082           │
   │Cycle CRUD   │              │ qBittorrent API  │             │Playwright + YouTube│
   └─────────────┘              └──────────────────┘             └────────────────────┘

   ┌─────────────┐              ┌──────────────────┐             ┌────────────────────┐
   │audio-process│              │ image-processor  │             │   ai-services      │
   │   :8083     │              │    :8084         │             │    :8085           │
   │JavaCV audio │              │ Java AWT covers  │             │OpenAI + Ollama     │
   └─────────────┘              └──────────────────┘             └────────────────────┘

   ┌─────────────┐              ┌──────────────────┐             ┌────────────────────┐
   │telegram-bot │              │ video-creator    │             │ descriptions       │
   │   :8086     │              │    :8087         │             │    :8089           │
   │Send + manage│              │ JavaCV video     │             │Jsoup + AI summary  │
   └─────────────┘              └──────────────────┘             └────────────────────┘

   ┌─────────────┐
   │boosty-poster│
   │   :8090     │
   │ Playwright  │
   └─────────────┘

                        ┌─────────────────────────────────────────────┐
                        │              foundation (shared jar)         │
                        │  Cycle entity · DTOs · Repositories         │
                        │  FFmpegOperations (JavaCV) · WebClientConfig │
                        └─────────────────────────────────────────────┘

                        ┌──────────────────┐   ┌──────────────────────┐
                        │ audio_silence    │   │       whisper        │
                        │ service :8000    │   │      :8002           │
                        │ Python / Demucs  │   │  Python / Whisper    │
                        └──────────────────┘   └──────────────────────┘
```

### The Cycle — Central Data Model

One Cycle = one audiobook from search to publish.

```
Cycle
  id, cycleIdentifier, title, author, status
  └── CycleStatus: PENDING → SEARCHING → DOWNLOADING → PROCESSING_AUDIO
                   → PROCESSING_IMAGE → PROCESSING_VIDEO → POSTING
                   → COMPLETED | FAILED

  AudioPart[]     — raw chapters path, concatenated path, compressed path, duration
  ImagePart[]     — base AI cover, final cover with text overlay
  VideoPart[]     — final MP4 path
  TorrentTask[]   — hash, download path, qBittorrent status
  TelegramMediaGroup[] → TelegramMediaItem[] — message IDs, file IDs for re-use
```

Every service reads/writes Cycle state through `core-service`. No service stores state in memory or JSON files.

---

## 6. Key Technical Decisions

### Decision 1: JavaCV replaces FFmpeg CLI
**Problem:** `ProcessBuilder("ffmpeg", ...)` requires FFmpeg binary installed in every container. Railway and Docker environments need `apt-get install ffmpeg` — fragile, version-inconsistent, not portable.

**Decision:** Use **JavaCV + Bytedeco FFmpeg presets**. FFmpeg native libraries are bundled as Maven JAR dependencies — platform-specific classifiers (`linux-x86_64` for Docker/Railway, `windows-x86_64` for local dev). Zero external binary dependency.

**What this unlocks:**
- Deploy to Railway with a plain `FROM eclipse-temurin:17-jre-alpine` — no FFmpeg install step
- Exact FFmpeg version pinned in `pom.xml` — consistent across all environments
- Frame-level callbacks for real progress tracking
- Proper Java exceptions instead of exit-code + stderr parsing
- Fix for the concurrent `filePath.txt` bug — concatenation streams directly, no temp file

### Decision 2: PostgreSQL replaces JSON file storage
**Problem:** JSON files have no transactions, no query capability, and the entire file is read on every write. Concurrent cycles corrupt shared files.

**Decision:** All metadata in PostgreSQL via Spring Data JPA. Physical media files stay on disk; their paths are stored as DB columns. Flyway manages schema.

### Decision 3: Maven multi-module parent
**Problem:** Each service is an independent Spring Boot project today with duplicated dependency management.

**Decision:** Single Maven parent POM (`audiobook-automation-platform/pom.xml`) with `dependencyManagement` for all versions. Services are modules. `foundation` is a shared `jar` dependency.

### Decision 4: Spring Cloud Gateway as unified entry point
All traffic enters at `:8088` (current monolith port — no client change needed). Gateway routes to internal services on their individual ports.

---

## 7. Opportunity Map

Using the Opportunity Solution Tree structure — each opportunity maps to the services that solve it.

```
OUTCOME: Reliable, fully automated audiobook pipeline with zero manual intervention

├── OPP A: Job state visibility (nobody knows where a cycle is or why it failed)
│   ├── Solution: Cycle entity in core-service with status enum
│   ├── Solution: Admin frontend shows Cycle list with stage + logs
│   └── Solution: Telegram notifications on stage change

├── OPP B: Deployment fragility (FFmpeg CLI, port conflicts, manual config)
│   ├── Solution: JavaCV — FFmpeg in the JAR, no binary install
│   ├── Solution: Maven multi-module — one build, all services
│   └── Solution: Docker Compose for local; Railway for prod

├── OPP C: Concurrent cycle corruption (filePath.txt, JSON race conditions)
│   ├── Solution: Per-cycle directory structure, no shared temp files
│   ├── Solution: JavaCV streaming concatenation (no filePath.txt)
│   └── Solution: PostgreSQL replaces JSON file storage

├── OPP D: Distribution coverage (manual Boosty/YouTube/Telegram steps)
│   ├── Solution: boosty-poster-service (Playwright, auth state cached)
│   ├── Solution: YouTube upload via Google API client (already exists)
│   └── Solution: telegram-bot-service sends audio + cover as media group

└── OPP E: Content sourcing quality (scraper breaks when site changes)
    ├── Solution: content-search-service — isolated, easy to update selectors
    └── Solution: Multiple source adapters (baza-knig, YouTube, future sources)
```

---

## 8. Assumptions to Test

| Assumption | Risk | Test |
|---|---|---|
| JavaCV works correctly on Railway's Linux x86_64 containers | HIGH — if GPU codec calls fail silently, audio/video output is silent/broken | Build minimal Docker image with JavaCV, run a transcode, assert output duration matches input |
| Spring Cloud Gateway adds no meaningful latency for internal calls | LOW — all services are on the same network | Benchmark gateway vs direct call on a health endpoint |
| Cycle-scoped directories eliminate the concurrent corruption bug | MEDIUM — race on directory creation if two cycles start simultaneously | Load test: start 5 cycles in parallel, verify no cross-contamination |
| Playwright Boosty auth state survives between deployments | HIGH — if token expires mid-deploy, posting stops | Test: deploy a new container, verify Boosty posting works without re-login |
| `foundation` as a shared JAR does not cause circular dependency issues | LOW — foundation has no service deps | Compile check during Phase 1 |

---

## 9. Phased Execution Roadmap

### Phase 0 — Stabilise monolith (this week)
**Goal:** Green tests, clean repo, no surprises before refactor begins.
- [ ] Remove `node/` binary folder from Java source tree (100MB bloat)
- [ ] Confirm all tests pass with `./mvnw test -Dspring.profiles.active=dev`
- [ ] Tag `v0-monolith` in git
- [ ] Document all env vars used (validate `env.example` is complete)

### Phase 1 — Maven parent + foundation (Week 1–2)
**Goal:** The shared scaffolding everything else plugs into.
- [ ] Create `audiobook-automation-platform/pom.xml` (parent, `packaging=pom`)
- [ ] Create `foundation/` module — enums, entities, DTOs, repositories, `FFmpegOperations` (JavaCV), `WebClientConfig`
- [ ] Validate: `./mvnw compile` from parent succeeds
- [ ] Write entity unit tests (Cycle status transitions)

**Deliverable:** `foundation-1.0.0-SNAPSHOT.jar` — importable by all services

### Phase 2 — Core service + API Gateway (Week 2–3)
**Goal:** You can start and track a Cycle via REST.
- [ ] `core-service` — `CycleController`, `CycleServiceImpl`, Flyway migrations
- [ ] `api-gateway-service` — Spring Cloud Gateway, routes for all future services
- [ ] Integration test: create Cycle via gateway → core-service responds
- [ ] Admin frontend connects to gateway (update `VITE_CORE_API_URL`)

**Deliverable:** Cycle CRUD live at `localhost:8088/api/cycles`

### Phase 3 — Self-contained service extractions (Week 3–6)
Extract in dependency order:

| Week | Service | Key work |
|---|---|---|
| 3 | `torrent-downloader-service` | Move qBittorrent logic, no other service deps |
| 3 | `audio-processing-service` | JavaCV audio ops replaces ProcessBuilder calls |
| 4 | `image-processor-service` | Java AWT cover generation, isolated |
| 4 | `ai-services` | OpenAI + Ollama clients, purely outbound HTTP |
| 5 | `descriptions-service` | Jsoup + WebClient calls to ai-services |
| 5 | `content-search-service` | Playwright + YouTube API, most complex scraper |
| 6 | `video-creator-service` | JavaCV video creation replaces 2-step FFmpeg CLI |

**Rule for each extraction:**
1. Create module under Maven parent
2. Copy relevant classes from monolith
3. Wire `foundation` dependency
4. Write `application.yml` with correct port
5. Integration test hits the actual endpoint
6. Delete extracted code from monolith
7. Deploy to Railway, verify health endpoint

### Phase 4 — Distribution services (Week 6–8)
- [ ] `telegram-bot-service` — bot registration, media group sender, message ID storage
- [ ] `boosty-poster-service` — Playwright auth state, post creation, file upload
- [ ] YouTube upload wired through `content-search-service` or own module
- [ ] End-to-end smoke test: one real cycle from scrape → all three platforms published

### Phase 5 — Harden (Week 8+, ongoing)
- [ ] Add Resilience4j retry + circuit breaker on all inter-service calls
- [ ] Proper Flyway migrations replace `ddl-auto=update`
- [ ] Distributed tracing (Spring Cloud Sleuth or Micrometer Tracing)
- [ ] Docker Compose that starts all services with a single command
- [ ] Secrets via Railway environment variables (remove all hardcoded fallbacks)

---

## 10. What We Are Explicitly Not Building (For Now)

- **Message queue (Kafka/RabbitMQ):** Services call each other via WebClient REST. A queue can be added later if retry/replay requirements grow — not needed to launch.
- **Kubernetes:** Docker Compose for local, Railway for prod. Kubernetes adds operational complexity that is not justified yet.
- **Consumer-facing features:** No public API, no user auth, no sign-up flow. Internal tool only.
- **Facebook Graph API integration:** Discussed in planning documents but not prioritised for the core pipeline.
- **Fabric by Daniel Miessler:** Interesting for YouTube transcript analysis but not on the critical path.

---

## 11. Open Questions

1. **Where does `boosty-poster-service` live?** The Playwright code for Boosty posting is the most operationally fragile part — auth state expires, selectors break when Boosty updates. Should it be isolated in its own service with a clear contract, or bundled with `content-search-service`? Recommendation: own service, makes it easier to re-auth without touching anything else.

2. **GPU-accelerated video on Railway?** `h264_nvenc` requires NVIDIA GPU. Railway standard instances are CPU-only. Do we need GPU plans, or is CPU-based H264 encoding acceptable for the video output quality needed? If CPU is fine, JavaCV with `AV_CODEC_ID_H264` + software encoder works without any GPU config.

3. **Cycle trigger mechanism?** How is a new cycle started today? Telegram command? REST call? This should be the first thing `core-service` exposes and the admin frontend supports.

---

## 12. Success Metrics

| Metric | Current | Target (3 months) |
|---|---|---|
| Time to publish one audiobook | Several hours manual | < 15 min end-to-end automated |
| Concurrent cycles possible | 1 (corrupt beyond that) | 5+ without interference |
| Deployment to new server | Manual FFmpeg install + config | `git push` → Railway deploys |
| Failed job recoverability | Manual restart from scratch | Resume from last completed stage |
| Services deployable independently | No (one monolith) | Yes (each service has own Railway service) |

---

*This plan lives at `docs/superpowers/specs/platform-product-plan.md`.
Update it as decisions are made and phases complete.*
