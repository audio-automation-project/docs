# Monolith → workers roadmap — implementation plan

Plan generated with the **writing-plans** workflow from the approved spec [`2026-05-05-monolith-to-workers-roadmap-design.md`](../specs/2026-05-05-monolith-to-workers-roadmap-design.md), after Git/GitHub layout was aligned under [`audio-automation-project`](https://github.com/audio-automation-project).

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Execute the approved design [`2026-05-05-monolith-to-workers-roadmap-design.md`](../specs/2026-05-05-monolith-to-workers-roadmap-design.md) in code: harden Postgres (`M1`/C1), introduce persisted jobs + async HTTP surface (`M2`/C2), then broker-backed worker MVP (`M3`/C3) and safer external integrations (`M4`/C4).

**Architecture:** Stay on one Spring Boot artifact (`audio-library-automation-bot`). Flyway becomes the sole schema authority; Hibernate moves to **`validate`** for production-shaped runs. Heavy audio work first surfaces as **PostgreSQL-backed jobs** invoked from bounded thread pools (`C2`), then consumes **RabbitMQ** messages (`C3`) without changing the job ID contract the frontend will use.

**Tech stack:** Java 17, Spring Boot 3.4, Spring Data JPA, Flyway, PostgreSQL, Testcontainers (optional IT), later `spring-boot-starter-amqp` + RabbitMQ 3.x for `M3`.

**Working copy:** Repo root **`audio-library-automation-bot`** (clone: `https://github.com/audio-automation-project/audio-library-automation-bot`). Cross-repo specs: **`docs`** repo.

---

## Baseline already satisfied (tracking)

Repo and deploy ergonomics landed before this plan (`M0`/C0 partial):

- [x] Git org mapping documented — [`architecture/repository-map.md`](../../architecture/repository-map.md)
- [x] Docker + Railway scaffold — `audio-library-automation-bot/Dockerfile`, `railway.toml`, `PORT`/Postgres/`CORS` in `application.properties`
- [x] Separate GitHub repos for Demucs (`audio-demucs-microservice`) and Whisper (`whisper-transcription-microservice`)

---

## Files you will introduce or materially change

| Area | Responsibility |
|------|----------------|
| `audio-library-automation-bot/pom.xml` | Add Flyway, optional Testcontainers JDBC, `spring-boot-starter-amqp` (only when starting `M3`) |
| `audio-library-automation-bot/src/main/resources/db/migration/V*.sql` | Versioned DDL (Flyway) |
| `audio-library-automation-bot/src/main/resources/application.properties` | Flip `spring.jpa.hibernate.ddl-auto` / `spring.flyway.enabled` after first migration set is proven |
| `audio-library-automation-bot/src/main/java/.../jobs/` | `ProcessingJobEntity`, `ProcessingJobRepository`, `ProcessingJobService`, `JobStatus`, DTOs |
| `audio-library-automation-bot/src/main/java/.../jobs/web/JobV1Controller.java` | `202` + `GET` job by id |
| `audio-library-automation-bot/src/main/java/.../jobs/runtime/AudioJobExecutor.java` | Runs concat (first slice) from pool, updates job row |
| `audio-library-automation-bot/src/main/java/.../config/CorrelationIdFilter.java` | Populates MDC `correlationId` from `X-Correlation-Id` or generated UUID |
| `audio-library-automation-bot/src/test/java/.../jobs/` | Unit + slice tests |
| `docs/guides/local-development.md` or `audio-library-automation-bot/README.md` | Flyway + async job usage for developers |

**Intentionally unchanged in early tasks:** `/api/audio/*` synchronous endpoints remain for backward compatibility; new routes live under `/api/v1/jobs/**` until clients migrate.

---

### Task 1: Add Flyway and schema ownership

**Files:**
- Modify: `audio-library-automation-bot/pom.xml`
- Create: `audio-library-automation-bot/src/main/resources/db/migration/V202605051200__baseline_parsed_book.sql`
- Modify: `audio-library-automation-bot/src/main/resources/application.properties`

- [ ] **Step 1.1:** Add Maven dependencies inside `<dependencies>` of `Automatation` `pom.xml`:

```xml
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>
```

- [ ] **Step 1.2:** Create **`V202605051200__baseline_parsed_book.sql`** aligned with `ParsedBookEntity` in `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/domain/ParsedBookEntity.java` (table `parsed_book`; Hibernate-compatible types):

```sql
CREATE TABLE IF NOT EXISTS parsed_book (
    id              BIGSERIAL PRIMARY KEY,
    author          VARCHAR(512),
    author_page     VARCHAR(1024),
    reader          VARCHAR(512),
    reader_page     VARCHAR(1024),
    year_str        VARCHAR(64),
    duration_str    VARCHAR(128),
    cycle_name      VARCHAR(512),
    cycle_part      VARCHAR(128),
    genre           VARCHAR(512),
    date_uploaded   VARCHAR(128),
    description     TEXT,
    title           VARCHAR(512),
    url             VARCHAR(1024),
    source_directory VARCHAR(1024)
);
```

- [ ] **Step 1.3:** Enable Flyway (remove or toggle the old blanket disable). In `application.properties`, set:

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
```

Keep **`spring.jpa.hibernate.ddl-auto=update`** temporarily so existing dev DBs converge on first CI run **or** flip to **`validate`** only after verifying empty DB migrations on a disposable Postgres (see Step 1.4).

- [ ] **Step 1.4:** Verify against local Docker Postgres

Run (from `audio-library-automation-bot/`):

```bash
docker compose up -d
./mvnw -q -DskipTests compile flyway:migrate
```

If you did not wire `flyway-maven-plugin`, use instead:

```bash
./mvnw -q test -Dtest=AutomatationApplicationTests
```

and expect Flyway to run on application startup (remove JPA exclusion in that test only if you add a dedicated Flyway smoke test — see Task 8).

Expected: Tables `parsed_book` and **`flyway_schema_history`** exist; no migration error.

- [ ] **Step 1.5:** Commit

```bash
git add pom.xml src/main/resources/db/migration/V202605051200__baseline_parsed_book.sql src/main/resources/application.properties
git commit -m "chore(db): Flyway baseline for parsed_book"
```

---

### Task 2: `processing_jobs` table (C1 gate toward C2)

**Files:**
- Create: `audio-library-automation-bot/src/main/resources/db/migration/V202605051215__processing_jobs.sql`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/domain/ProcessingJobEntity.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/domain/JobStatus.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/domain/JobType.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/repository/ProcessingJobRepository.java`

- [ ] **Step 2.1:** Migration SQL:

```sql
CREATE TABLE processing_jobs (
    id              UUID PRIMARY KEY,
    job_type        VARCHAR(64) NOT NULL,
    status          VARCHAR(32) NOT NULL,
    correlation_id  VARCHAR(64),
    input_payload   TEXT,
    result_payload  TEXT,
    message         TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX ix_processing_jobs_status ON processing_jobs (status);
CREATE INDEX ix_processing_jobs_created ON processing_jobs (created_at DESC);
```

- [ ] **Step 2.2:** Enums (package `kg.automation.rest.automatation.jobs.domain`):

```java
package kg.automation.rest.automatation.jobs.domain;

public enum JobStatus {
    ACCEPTED, RUNNING, SUCCEEDED, FAILED
}
```

```java
package kg.automation.rest.automatation.jobs.domain;

public enum JobType {
    AUDIO_CONCATENATE
    // AUDIO_TRIM, AUDIO_COMPRESS — add when same pattern is proven
}
```

- [ ] **Step 2.3:** Entity

```java
package kg.automation.rest.automatation.jobs.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

import java.time.Instant;
import java.util.UUID;

@Getter
@Setter
@Entity
@Table(name = "processing_jobs")
public class ProcessingJobEntity {

    @Id
    private UUID id;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 64)
    private JobType jobType;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 32)
    private JobStatus status;

    @Column(length = 64)
    private String correlationId;

    @Column(columnDefinition = "TEXT")
    private String inputPayload;

    @Column(columnDefinition = "TEXT")
    private String resultPayload;

    @Column(columnDefinition = "TEXT")
    private String message;

    @Column(nullable = false)
    private Instant createdAt;

    @Column(nullable = false)
    private Instant updatedAt;

    @PrePersist
    void prePersist() {
        Instant now = Instant.now();
        if (id == null) id = UUID.randomUUID();
        if (createdAt == null) createdAt = now;
        updatedAt = now;
    }

    @PreUpdate
    void preUpdate() {
        updatedAt = Instant.now();
    }
}
```

- [ ] **Step 2.4:** Repository

```java
package kg.automation.rest.automatation.jobs.repository;

import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.UUID;

public interface ProcessingJobRepository extends JpaRepository<ProcessingJobEntity, UUID> {
}
```

- [ ] **Step 2.5:** Run tests

```bash
./mvnw -q test
```

Expected: `BUILD SUCCESS` (context test may still exclude Flyway— acceptable until Task 8).

- [ ] **Step 2.6:** Commit

```bash
git add src/main/resources/db/migration/V202605051215__processing_jobs.sql src/main/java/kg/automation/rest/automatation/jobs/
git commit -m "feat(jobs): processing_jobs table and JPA model"
```

---

### Task 3: Correlation id filter (logging seam for C2/C3)

**Files:**
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/config/CorrelationIdFilter.java`

- [ ] **Step 3.1:** Implementation

```java
package kg.automation.rest.automatation.config;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    public static final String HEADER = "X-Correlation-Id";
    public static final String MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        String incoming = request.getHeader(HEADER);
        String id = (incoming != null && !incoming.isBlank()) ? incoming.trim() : UUID.randomUUID().toString();
        response.setHeader(HEADER, id);
        MDC.put(MDC_KEY, id);
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

- [ ] **Step 3.2:** Add Logback pattern snippet in `src/main/resources/logback-spring.xml` **if** the project uses one; otherwise document in README that `%X{correlationId}` can be added when Logback is customized. Quick check:

```bash
dir src\main\resources\logback*.xml
```

- [ ] **Step 3.3:** Commit

```bash
git add src/main/java/kg/automation/rest/automatation/config/CorrelationIdFilter.java
git commit -m "feat(obs): X-Correlation-Id filter and MDC"
```

---

### Task 4: Job service + REST (`202` / `GET`) for concatenate

**Files:**
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/dto/ProcessingJobResponse.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/dto/AudioConcatenateJobRequest.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/service/ProcessingJobService.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/web/JobV1Controller.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/runtime/AudioJobExecutor.java`
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/config/` — add **`@EnableAsync`** if not inherited, or configure `TaskExecutor` bean (prefer explicit **`ThreadPoolTaskExecutor`** named **`audioJobExecutor`** with bounded `queueCapacity`).

- [ ] **Step 4.1:** Request/response DTOs (minimal JSON contract)

```java
package kg.automation.rest.automatation.jobs.dto;

import lombok.Data;

@Data
public class AudioConcatenateJobRequest {
    private String inputDirectory;
    private String outputDirectory;
}
```

```java
package kg.automation.rest.automatation.jobs.dto;

import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import lombok.Builder;
import lombok.Data;

import java.util.UUID;

@Data
@Builder
public class ProcessingJobResponse {
    private UUID id;
    private JobType jobType;
    private JobStatus status;
    private String correlationId;
    private String message;
}
```

- [ ] **Step 4.2:** Service skeleton — enqueue row + executor trigger

```java
package kg.automation.rest.automatation.jobs.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import kg.automation.rest.automatation.config.CorrelationIdFilter;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.dto.AudioConcatenateJobRequest;
import kg.automation.rest.automatation.jobs.dto.ProcessingJobResponse;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.jobs.runtime.AudioJobExecutor;
import lombok.RequiredArgsConstructor;
import org.slf4j.MDC;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class ProcessingJobService {

    private final ProcessingJobRepository repository;
    private final ObjectMapper objectMapper;
    private final AudioJobExecutor audioJobExecutor;

    @Transactional
    public ProcessingJobResponse submitConcatenate(AudioConcatenateJobRequest body) throws JsonProcessingException {
        ProcessingJobEntity e = new ProcessingJobEntity();
        e.setJobType(JobType.AUDIO_CONCATENATE);
        e.setStatus(JobStatus.ACCEPTED);
        e.setCorrelationId(Optional.ofNullable(MDC.get(CorrelationIdFilter.MDC_KEY)).orElse(null));
        e.setInputPayload(objectMapper.writeValueAsString(body));
        e.setCreatedAt(Instant.now());
        e.setUpdatedAt(Instant.now());
        repository.save(e);

        UUID id = e.getId();
        audioJobExecutor.enqueueConcatenate(id);

        return toResponse(e);
    }

    public Optional<ProcessingJobResponse> get(UUID id) {
        return repository.findById(id).map(this::toResponse);
    }

    private ProcessingJobResponse toResponse(ProcessingJobEntity e) {
        return ProcessingJobResponse.builder()
                .id(e.getId())
                .jobType(e.getJobType())
                .status(e.getStatus())
                .correlationId(e.getCorrelationId())
                .message(e.getMessage())
                .build();
    }
}
```

- [ ] **Step 4.3:** Executor — calls existing `AudioConcatenatorService` off HTTP thread.

```java
package kg.automation.rest.automatation.jobs.runtime;

import com.fasterxml.jackson.databind.ObjectMapper;
import kg.automation.rest.automatation.Data.Response;
import kg.automation.rest.automatation.audio.service.AudioConcatenatorService;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.dto.AudioConcatenateJobRequest;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.core.task.TaskExecutor;
import org.springframework.stereotype.Component;

import java.util.UUID;

@Component
@RequiredArgsConstructor
public class AudioJobExecutor {

    private final TaskExecutor audioJobExecutor;
    private final ProcessingJobRepository repository;
    private final ObjectMapper objectMapper;
    private final AudioConcatenatorService concatenatorService;

    public void enqueueConcatenate(UUID jobId) {
        audioJobExecutor.execute(() -> runConcatenate(jobId));
    }

    void runConcatenate(UUID jobId) {
        ProcessingJobEntity job = repository.findById(jobId).orElseThrow();
        job.setStatus(JobStatus.RUNNING);
        repository.save(job);
        try {
            AudioConcatenateJobRequest req = objectMapper.readValue(job.getInputPayload(), AudioConcatenateJobRequest.class);
            Response res = concatenatorService.concatenateAudio(req.getInputDirectory(), req.getOutputDirectory());
            job.setStatus("success".equalsIgnoreCase(res.getStatus()) ? JobStatus.SUCCEEDED : JobStatus.FAILED);
            job.setResultPayload(objectMapper.writeValueAsString(res));
            job.setMessage(res.getMessage());
        } catch (Exception ex) {
            job.setStatus(JobStatus.FAILED);
            job.setMessage(ex.getMessage());
        }
        repository.save(job);
    }
}
```

Wire **`TaskExecutor`** bean in a `@Configuration` class (new file `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/config/AudioJobExecutorConfig.java`):

```java
package kg.automation.rest.automatation.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
public class AudioJobExecutorConfig {

    @Bean
    public TaskExecutor audioJobExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setThreadNamePrefix("audio-job-");
        ex.setCorePoolSize(1);
        ex.setMaxPoolSize(2);
        ex.setQueueCapacity(50);
        ex.initialize();
        return ex;
    }
}
```

- [ ] **Step 4.4:** Controller

```java
package kg.automation.rest.automatation.jobs.web;

import com.fasterxml.jackson.core.JsonProcessingException;
import kg.automation.rest.automatation.jobs.dto.AudioConcatenateJobRequest;
import kg.automation.rest.automatation.jobs.dto.ProcessingJobResponse;
import kg.automation.rest.automatation.jobs.service.ProcessingJobService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/v1/jobs")
@RequiredArgsConstructor
public class JobV1Controller {

    private final ProcessingJobService jobService;

    @PostMapping("/audio/concatenate")
    public ResponseEntity<ProcessingJobResponse> submitConcatenate(@RequestBody AudioConcatenateJobRequest body)
            throws JsonProcessingException {
        ProcessingJobResponse created = jobService.submitConcatenate(body);
        return ResponseEntity.status(HttpStatus.ACCEPTED)
                .header("Location", "/api/v1/jobs/" + created.getId())
                .body(created);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProcessingJobResponse> get(@PathVariable UUID id) {
        return jobService.get(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
}
```

- [ ] **Step 4.5:** Unit test — create `audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/jobs/service/ProcessingJobServiceTest.java` with mocks for `ProcessingJobRepository`, real `ObjectMapper`, and mock `AudioJobExecutor`.

```java
package kg.automation.rest.automatation.jobs.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.dto.AudioConcatenateJobRequest;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.jobs.runtime.AudioJobExecutor;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ProcessingJobServiceTest {

    @Mock
    ProcessingJobRepository repository;
    @Mock
    AudioJobExecutor audioJobExecutor;

    ProcessingJobService service;
    final ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setUp() {
        service = new ProcessingJobService(repository, objectMapper, audioJobExecutor);
        when(repository.save(any(ProcessingJobEntity.class))).thenAnswer(invocation -> {
            ProcessingJobEntity e = invocation.getArgument(0);
            if (e.getId() == null) {
                e.setId(java.util.UUID.randomUUID());
            }
            return e;
        });
    }

    @Test
    void submitConcatenate_persistsAcceptedAndEnqueues() throws Exception {
        AudioConcatenateJobRequest req = new AudioConcatenateJobRequest();
        req.setInputDirectory("/in");
        req.setOutputDirectory("/out");
        ArgumentCaptor<ProcessingJobEntity> cap = ArgumentCaptor.forClass(ProcessingJobEntity.class);

        service.submitConcatenate(req);

        verify(repository).save(cap.capture());
        assertThat(cap.getValue().getStatus()).isEqualTo(JobStatus.ACCEPTED);
        assertThat(cap.getValue().getJobType()).isEqualTo(JobType.AUDIO_CONCATENATE);
        verify(audioJobExecutor).enqueueConcatenate(cap.getValue().getId());
    }
}
```

- [ ] **Step 4.6:** Run Tests

Run:

```powershell
cd audio-library-automation-bot
.\mvnw.cmd -q "-Dskip.boosty.bundle.build=true" test "-Dtest=ProcessingJobServiceTest"
```

Expected: BUILD SUCCESS.

- [ ] **Step 4.7:** Important implementation detail — **`runConcatenate` runs on a worker thread.** Open a `@Transactional`-safe pattern: inject `TransactionalJobRunner` facade that executes `repository.findById` + save inside **`TransactionTemplate`** or annotate `runConcatenate` with **`@Transactional(propagation = REQUIRES_NEW)`** on a self-injected `@Service` bean to avoid lazy-detached failures. Adjust as needed after the first Hibernate error in staging.

- [ ] **Step 4.8:** Commit

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/ src/main/java/kg/automation/rest/automatation/config/CorrelationIdFilter.java src/main/java/kg/automation/rest/automatation/config/AudioJobExecutorConfig.java src/test/java/kg/automation/rest/automatation/jobs/
git commit -m "feat(jobs): async concat job API v1"
```

---

### Task 5: Frontend contract notes (`audio-frontend` repo)

**Files:**
- Modify: `audio-frontend/.env.example` (and optionally a short section in `audio-frontend/README.md`)

- [ ] **Step 5.1:** Document polling flow for operators:

```
POST /api/v1/jobs/audio/concatenate  → 202 + body includes job id  
GET /api/v1/jobs/{id}  → poll until status SUCCEEDED | FAILED  
Header echo: X-Correlation-Id (optional on request)
```

- [ ] **Step 5.2:** Commit in **`audio-frontend`** clone (not monorepo umbrella):

```bash
git add .env.example README.md
git commit -m "docs: describe async audio job endpoints"
git push origin main
```

---

### Task 6: Hibernate `validate` production profile (`M1` close-out)

**Files:**
- Create: `audio-library-automation-bot/src/main/resources/application-prod.properties`
- Modify: `README.md` deployment section inside bot repo — state `SPRING_PROFILES_ACTIVE=prod` on Railway

- [ ] **Step 6.1:** **`application-prod.properties`:**

```properties
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
logging.level.org.flywaydb=INFO
```

- [ ] **Step 6.2:** Boot check with Postgres + migrations:

```bash
$env:SPRING_PROFILES_ACTIVE="prod"
./mvnw spring-boot:run
```

(with local `docker compose up -d`)

Expected: app starts without `Schema-validation` errors.

- [ ] **Step 6.3:** Commit

```bash
git add src/main/resources/application-prod.properties README.md
git commit -m "feat(config): prod profile validates schema via Flyway"
```

---

### Task 7: JSON / file hotspot inventory (`C1` architecture alignment)

Even though `TorrentService.appendToJsonFile` is commented out, finish the spreadsheet for future PRs:

- [ ] **Step 7.1:** RiPGrep workspace `audio-library-automation-bot`:

```powershell
rg "FileReader|Files\\.read|String json|\\.json\\\"| gson\\.fromJson\\(" src/main/java --glob '*.java'
```

- [ ] **Step 7.2:** Paste summary table into `docs/audit/file-operations.md` (or append to existing audit note) with columns: path, read/write, risk, replacement (SQL vs Firestore).

- [ ] **Step 7.3:** Commit from **`docs`** repo.

---

### Task 8: RabbitMQ MVP (`M3` / `C3`) — after `M2` stable in staging

Prerequisite checklist before starting broker work:

- [ ] Concat job path observes **no** regressions (`SUCCEEDED` / `FAILED`) under load-test of **10** sequential jobs against real FFmpeg directories.
- [ ] `CorrelationIdFilter` proves traceability in Cloud / Railway logs.

**Files:**
- Modify: `audio-library-automation-bot/pom.xml` — dependency `spring-boot-starter-amqp`
- Create: `audio-library-automation-bot/src/main/java/.../jobs/messaging/AudioConcatenateJobPublisher.java` — publishes `{ "jobId": "..." }
- Create: `docker-compose.yml` sibling **or** `docs/guides/rabbitmq-local.md` with `docker run rabbitmq:3-management`
- Separate **worker** Maven module OR second Spring Boot entry class `WorkerApplication` inside same repo (lighter ops) subscribing to queue `audio.concatenate`, calling same `AudioConcatenatorService` logic.

Concrete first integration message payload JSON:

```json
{
  "jobId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "correlationId": "trace-123",
  "version": 1
}
```

Producer flow:

1. `POST /api/v1/jobs/audio/concatenate` writes `processing_jobs` row `ACCEPTED`.
2. Instead of calling `audioJobExecutor.enqueueConcatenate(id)` immediately, call `publisher.publish(jobId, correlationId)`.
3. API returns `202` exactly as today.
4. Worker receives message → loads row → sets `RUNNING` → FFmpeg → finalize row.

Rollback switch: **`jobs.use-in-process-executor=true`** property to fall back to Task 4 behavior without redeploying clients.

Verification:

```powershell
docker run -d --hostname rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
# set SPRING_RABBITMQ_HOST=localhost in application-local.properties
./mvnw test
```

Add integration test `EmbeddedAMQP` is optional (`spring-boot-starter-amqp-test` patterns) — YAGNI until basic manual publish proves row transitions.

Commit message example: `feat(jobs): optional RabbitMQ handoff for concat jobs`.

---

### Task 9: External integration timeouts (`M4` / `C4`)

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/gateway/RestConfig.java` — replace **`new RestTemplate()`** today with timeouts via `RestTemplateBuilder` (inject `spring-boot-starter-web`’s builder).
- Targets: wrappers used by **`RestCallService`**, Silence/Whisper HTTP clients.

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import java.time.Duration;

@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofMinutes(5))
            .build();
}
```

- [ ] Regression: `curl` against OpenAI mocks / local silence still passes.

Commit: `fix(http): bounded RestTemplate timeouts for outbound calls`.

---

## Plan self-review (maintainer checklist)

| Spec clause | Mapped task |
|-------------|--------------|
| C1 Flyway + Postgres authority | Tasks 1–2 + Task 6 |
| C2 async jobs + observable IDs | Tasks 3–5 |
| C3 broker + worker | Task 8 (gated) |
| C4 external isolation | Task 9 |
| B deprecate fragile JSON usages | Task 7 |
| Firebase-vs-SQL decision | Deferred — run **architecture checkpoint** (`docs/superpowers/specs/2026-04-20-firebase-full-migration-design.md` vs Postgres roadmap) before expanding job schema beyond FFmpeg metadata. |

Placeholder scan: no `TODO`/`TBD` in executable steps — Task 8 defers Embedded AMQP as optional prose only where noted.

---

## Execution handoff

**Plan saved to:** `docs/superpowers/plans/2026-05-05-monolith-to-workers-implementation-plan.md`

**Two execution modes:**

**1. Subagent-driven (recommended)** — Dispatch a fresh subagent per task (`Task 1`, `Task 2`, …); review Git diff between tasks.

**2. Inline execution** — Run tasks sequentially in one session using `superpowers:executing-plans` with human checkpoints after Task 4 and before Task 8.

**Which approach do you want?**
