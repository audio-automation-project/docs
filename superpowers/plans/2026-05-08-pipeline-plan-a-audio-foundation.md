# Pipeline Orchestrator — Plan A: Audio Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the complete audio processing chain (noise removal → trim → concatenate → compress → duration) as RabbitMQ-dispatched `job_queue` jobs with a `PipelineOrchestrator` that advances audio phases by querying DB state.

**Architecture:** Each audio job type owns exactly one `audio_part_id` FK. A `PipelineOrchestrator` checks completed-job counts against expected counts per cycle and emits the next job batch. Audio runners follow a strict pattern: verify PENDING → mark IN_PROGRESS → do work → write `audio_stage` row → mark COMPLETED/FAILED. Python microservices (Demucs, Whisper) are called synchronously via HTTP from runners.

**Tech Stack:** Spring Boot 3.4, Java 17, Spring AMQP, Flyway, JPA, FFmpeg/JavaCV (`FFmpegOperations`), `RestTemplate` (HTTP to Python microservices).

---

## File map

**New files:**
- `src/main/resources/db/migration/V202605080001__pipeline_run.sql` — `pipeline_run` table + `pipeline_run_id` FK on `job_queue`
- `src/main/java/.../jobs/domain/PipelineRunStatus.java` — enum: PENDING, RUNNING, COMPLETED, FAILED, CANCELLED
- `src/main/java/.../jobs/domain/PipelineRunEntity.java` — JPA entity for `pipeline_run`
- `src/main/java/.../jobs/repository/PipelineRunRepository.java` — JPA repository
- `src/main/java/.../gateway/SilenceServiceClient.java` — HTTP client for Demucs noise removal (:8000)
- `src/main/java/.../gateway/WhisperServiceClient.java` — HTTP client for Whisper transcription (:8002)
- `src/main/java/.../jobs/service/AudioNoiseRemoveJobRunner.java` — AUDIO_NOISE_REMOVE runner
- `src/main/java/.../jobs/service/AudioTrimJobRunner.java` — AUDIO_TRIM runner (FFmpeg silence remove)
- `src/main/java/.../jobs/service/AudioCompressJobRunner.java` — AUDIO_COMPRESS runner (one file, not a directory scan)
- `src/main/java/.../jobs/service/AudioDurationJobRunner.java` — AUDIO_DURATION runner (probe + write)
- `src/main/java/.../pipeline/orchestrator/PipelineOrchestrator.java` — phase-transition logic (audio phases only)
- `src/test/java/.../jobs/service/AudioNoiseRemoveJobRunnerTest.java`
- `src/test/java/.../jobs/service/AudioTrimJobRunnerTest.java`
- `src/test/java/.../jobs/service/AudioCompressJobRunnerTest.java`
- `src/test/java/.../jobs/service/AudioDurationJobRunnerTest.java`
- `src/test/java/.../pipeline/orchestrator/PipelineOrchestratorAudioTest.java`

**Modified files:**
- `src/main/java/.../jobs/domain/JobType.java` — add AUDIO_NOISE_REMOVE, AUDIO_TRIM, AUDIO_COMPRESS, AUDIO_DURATION, IMAGE_GENERATE, VIDEO_RENDER, DISTRIBUTE, CLEANUP_CYCLE
- `src/main/java/.../jobs/domain/ProcessingJobEntity.java` — add `pipelineRunId` field
- `src/main/java/.../ffmpeg/FFmpegOperations.java` — add `silenceRemoveMp3(Path input, Path output)`
- `src/main/java/.../infra/rabbit/InfraRabbitConfiguration.java` — add `audio.noise-remove` dedicated queue
- `src/main/java/.../infra/rabbit/PipelineJobsRabbitListener.java` — dispatch AUDIO_NOISE_REMOVE, AUDIO_TRIM, AUDIO_COMPRESS, AUDIO_DURATION
- `src/main/java/.../jobs/runtime/AudioJobExecutor.java` — add `enqueue(JobType, UUID)` generic method
- `src/test/java/.../AutomatationApplicationTests.java` — add mocks for new beans

---

## Recommended task order

Follow this sequence — do **not** execute alphabetically:

```
1 → 2 → 3 → 7 → 4 → 5 → 6 → 8 → 9 → 10 → 11 → 12 → 13 → 14
```

Rationale:
- **Task 7 before Task 4**: the `audio.noise-remove` queue must exist before any runner wires a listener to it.
- **Task 4 before Task 8**: `AudioTrimJobRunner` calls `FFmpegOperations.silenceRemoveMp3`. The method must compile and the filter approach must be confirmed working before Task 8 depends on it.
- **Mocks are added per-task** (not all at once in Task 14) so `AutomatationApplicationTests.contextLoads()` stays green after every task.

---

## Task 1: Flyway — pipeline_run table and pipeline_run_id on job_queue

**Files:**
- Create: `src/main/resources/db/migration/V202605080001__pipeline_run.sql`

- [ ] **Step 1: Write the migration**

```sql
-- Pipeline run: groups all job_queue rows for one orchestrated invocation.
-- pipeline_run_id on job_queue enables per-run dashboards and retry isolation.

CREATE TABLE pipeline_run (
    id            UUID        NOT NULL DEFAULT gen_random_uuid(),
    cycle_id      BIGINT      NOT NULL REFERENCES cycle (id) ON DELETE CASCADE,
    status        TEXT        NOT NULL DEFAULT 'PENDING',
    current_phase TEXT,
    error_message TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at  TIMESTAMPTZ,
    CONSTRAINT pk_pipeline_run     PRIMARY KEY (id),
    CONSTRAINT chk_pipeline_run_status CHECK (
        status IN ('PENDING', 'RUNNING', 'COMPLETED', 'FAILED', 'CANCELLED')
    )
);

CREATE INDEX ix_pipeline_run_cycle  ON pipeline_run (cycle_id);
CREATE INDEX ix_pipeline_run_active ON pipeline_run (status) WHERE status IN ('PENDING', 'RUNNING');

CREATE TRIGGER trg_pipeline_run_updated_at
    BEFORE UPDATE ON pipeline_run
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Add pipeline_run grouping FK to job_queue
ALTER TABLE job_queue
    ADD COLUMN pipeline_run_id UUID REFERENCES pipeline_run (id);

CREATE INDEX ix_job_queue_run ON job_queue (pipeline_run_id)
    WHERE pipeline_run_id IS NOT NULL;
```

- [ ] **Step 2: Run tests to verify migration applies cleanly**

```bash
cd audio-library-automation-bot
./mvnw test -Dtest=FlywayCycleSchemaIT -pl .
```

Expected: BUILD SUCCESS (Testcontainers applies all migrations including V202605080001)

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/db/migration/V202605080001__pipeline_run.sql
git commit -m "feat: pipeline_run table + pipeline_run_id FK on job_queue"
```

---

## Task 2: PipelineRunEntity, PipelineRunStatus, PipelineRunRepository

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/jobs/domain/PipelineRunStatus.java`
- Create: `src/main/java/kg/automation/rest/automatation/jobs/domain/PipelineRunEntity.java`
- Create: `src/main/java/kg/automation/rest/automatation/jobs/repository/PipelineRunRepository.java`
- Modify: `src/main/java/kg/automation/rest/automatation/jobs/domain/ProcessingJobEntity.java`

- [ ] **Step 1: Create PipelineRunStatus enum**

```java
package kg.automation.rest.automatation.jobs.domain;

public enum PipelineRunStatus {
    PENDING,
    RUNNING,
    COMPLETED,
    FAILED,
    CANCELLED
}
```

- [ ] **Step 2: Create PipelineRunEntity**

```java
package kg.automation.rest.automatation.jobs.domain;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.PreUpdate;
import jakarta.persistence.Table;
import lombok.Getter;
import lombok.Setter;

import java.time.Instant;
import java.util.UUID;

@Getter
@Setter
@Entity
@Table(name = "pipeline_run")
public class PipelineRunEntity {

    @Id
    private UUID id;

    @Column(name = "cycle_id", nullable = false)
    private Long cycleId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 32)
    private PipelineRunStatus status;

    @Column(name = "current_phase", length = 64)
    private String currentPhase;

    @Column(name = "error_message", columnDefinition = "TEXT")
    private String errorMessage;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @Column(name = "completed_at")
    private Instant completedAt;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
        Instant now = Instant.now();
        if (createdAt == null) {
            createdAt = now;
        }
        updatedAt = now;
        if (status == null) {
            status = PipelineRunStatus.PENDING;
        }
    }

    @PreUpdate
    void preUpdate() {
        updatedAt = Instant.now();
    }
}
```

- [ ] **Step 3: Create PipelineRunRepository**

```java
package kg.automation.rest.automatation.jobs.repository;

import kg.automation.rest.automatation.jobs.domain.PipelineRunEntity;
import kg.automation.rest.automatation.jobs.domain.PipelineRunStatus;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface PipelineRunRepository extends JpaRepository<PipelineRunEntity, UUID> {

    List<PipelineRunEntity> findByCycleIdAndStatus(Long cycleId, PipelineRunStatus status);

    Optional<PipelineRunEntity> findFirstByCycleIdOrderByCreatedAtDesc(Long cycleId);
}
```

- [ ] **Step 4: Add pipelineRunId to ProcessingJobEntity**

Add one field after `distributionId` in `ProcessingJobEntity.java`:

```java
@Column(name = "pipeline_run_id")
private UUID pipelineRunId;
```

- [ ] **Step 5: Run context-loads test**

```bash
./mvnw test -Dtest=AutomatationApplicationTests
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/domain/PipelineRunStatus.java \
        src/main/java/kg/automation/rest/automatation/jobs/domain/PipelineRunEntity.java \
        src/main/java/kg/automation/rest/automatation/jobs/repository/PipelineRunRepository.java \
        src/main/java/kg/automation/rest/automatation/jobs/domain/ProcessingJobEntity.java
git commit -m "feat: PipelineRunEntity + repository + pipelineRunId on job_queue entity"
```

---

## Task 3: Expand JobType enum

**Files:**
- Modify: `src/main/java/kg/automation/rest/automatation/jobs/domain/JobType.java`

- [ ] **Step 1: Replace JobType with full set**

```java
package kg.automation.rest.automatation.jobs.domain;

public enum JobType {
    /** Existing: concatenates per-part source MP3s into one file. Owner: audio_part_id. */
    AUDIO_CONCATENATE,
    /** Calls Demucs noise-removal microservice. ORIGINAL → NOISE_REMOVED. Owner: audio_part_id. */
    AUDIO_NOISE_REMOVE,
    /** FFmpeg silence-trim filter. NOISE_REMOVED → TRIMMED. Owner: audio_part_id. */
    AUDIO_TRIM,
    /** Re-encodes to target bitrate. CONCATENATED → COMPRESSED. Owner: audio_part_id. */
    AUDIO_COMPRESS,
    /** Probes latest audio stage and writes audio_part.duration_seconds. Owner: audio_part_id. */
    AUDIO_DURATION,
    /** Generates cover image via BookCoverService and persists to image_asset. Owner: image_asset_id. */
    IMAGE_GENERATE,
    /** Renders video from audio + image. Owner: video_part_id. */
    VIDEO_RENDER,
    /** Posts to one platform; writes distribution_result. Owner: distribution_id. */
    DISTRIBUTE,
    /** Deletes cycle filesystem files after APPROVED_FOR_DELETE lifecycle gate. Owner: cycle_id only. */
    CLEANUP_CYCLE
}
```

- [ ] **Step 2: Verify compilation**

```bash
./mvnw compile -q
```

Expected: BUILD SUCCESS (no callers of JobType need changes yet; listener drops unknowns with a log)

- [ ] **Step 3: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/domain/JobType.java
git commit -m "feat: expand JobType enum with all production pipeline job types"
```

---

## Task 4: Add silenceRemoveMp3 to FFmpegOperations

**Files:**
- Modify: `src/main/java/kg/automation/rest/automatation/ffmpeg/FFmpegOperations.java`

- [ ] **Step 1: Write failing test for silenceRemoveMp3**

Add this test class:

```java
// src/test/java/kg/automation/rest/automatation/ffmpeg/FFmpegSilenceRemoveTest.java
package kg.automation.rest.automatation.ffmpeg;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class FFmpegSilenceRemoveTest {

    private final FFmpegOperations ops = new FFmpegOperations("cpu");

    @Test
    void silenceRemoveMp3_missingInput_throwsIllegalArgument(@TempDir Path tmp) {
        Path missing = tmp.resolve("missing.mp3");
        Path out = tmp.resolve("out.mp3");
        assertThatThrownBy(() -> ops.silenceRemoveMp3(missing, out))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("does not exist");
    }

    @Test
    void silenceRemoveMp3_emptyFile_throwsIllegalArgument(@TempDir Path tmp) throws IOException {
        Path empty = tmp.resolve("empty.mp3");
        Files.writeString(empty, "");
        Path out = tmp.resolve("out.mp3");
        assertThatThrownBy(() -> ops.silenceRemoveMp3(empty, out))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
./mvnw test -Dtest=FFmpegSilenceRemoveTest
```

Expected: FAIL — `silenceRemoveMp3 not found`

- [ ] **Step 3: Add silenceRemoveMp3 method to FFmpegOperations — primary approach**

Insert this method after `trimAudioToMp3` in `FFmpegOperations.java`. Start with the **recorder approach** (primary):

```java
/**
 * Removes leading/trailing silences from {@code input} and writes the result to {@code output}.
 * Uses the FFmpeg {@code silenceremove} filter chain via recorder audio options.
 * Input must exist and be non-empty. Output parent directory must exist.
 */
public void silenceRemoveMp3(Path input, Path output) {
    if (!Files.exists(input)) {
        throw new IllegalArgumentException("Input does not exist: " + input);
    }
    try {
        if (Files.size(input) == 0) {
            throw new IllegalArgumentException("Input file is empty: " + input);
        }
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
    try (FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(input.toFile())) {
        grabber.start();
        try (FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(output.toFile(), grabber.getAudioChannels())) {
            recorder.setAudioCodec(avcodec.AV_CODEC_ID_MP3);
            recorder.setAudioBitrate(grabber.getAudioBitrate() > 0 ? grabber.getAudioBitrate() : 128_000);
            recorder.setSampleRate(grabber.getSampleRate() > 0 ? grabber.getSampleRate() : 44_100);
            recorder.setAudioChannels(Math.max(grabber.getAudioChannels(), 1));
            recorder.setAudioOption("af",
                    "silenceremove=start_periods=1:start_silence=0.5:start_threshold=-50dB"
                    + ",areverse"
                    + ",silenceremove=start_periods=1:start_silence=0.5:start_threshold=-50dB"
                    + ",areverse");
            recorder.start();
            Frame frame;
            while ((frame = grabber.grabSamples()) != null) {
                recorder.record(frame);
            }
        }
    } catch (Exception e) {
        throw new UncheckedIOException("silenceRemoveMp3 failed: " + input, new IOException(e.getMessage(), e));
    }
}
```

Add `import org.bytedeco.javacv.Frame;` and `import org.bytedeco.javacv.FFmpegFrameGrabber;` if not already present.

- [ ] **Step 3b: Verify the recorder approach actually applies the filter**

`FFmpegFrameRecorder.setAudioOption("af", ...)` may silently ignore arbitrary filter chains — the option key is not guaranteed to wire into `avfilter`. Run the input-validation tests first, then manually smoke-test with a real MP3:

```bash
# Copy any small MP3 into /tmp/test_in.mp3, then run:
./mvnw test -Dtest=FFmpegSilenceRemoveTest
```

If the tests pass (input validation only), also verify the filter is applied by running the method manually in a quick test class against a real MP3 and confirming the output file exists and is shorter. Look for any log warnings containing `"af"` or `"silenceremove"` being unrecognised.

**If the recorder approach does not apply the filter** (output file is same length, or `setAudioOption` throws an unsupported-operation error), replace the method body with the **grabber approach** instead:

```java
public void silenceRemoveMp3(Path input, Path output) {
    if (!Files.exists(input)) {
        throw new IllegalArgumentException("Input does not exist: " + input);
    }
    try {
        if (Files.size(input) == 0) {
            throw new IllegalArgumentException("Input file is empty: " + input);
        }
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
    try (FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(input.toFile())) {
        grabber.setOption("af",
                "silenceremove=start_periods=1:start_silence=0.5:start_threshold=-50dB"
                + ",areverse"
                + ",silenceremove=start_periods=1:start_silence=0.5:start_threshold=-50dB"
                + ",areverse");
        grabber.start();
        try (FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(output.toFile(), grabber.getAudioChannels())) {
            recorder.setAudioCodec(avcodec.AV_CODEC_ID_MP3);
            recorder.setAudioBitrate(grabber.getAudioBitrate() > 0 ? grabber.getAudioBitrate() : 128_000);
            recorder.setSampleRate(grabber.getSampleRate() > 0 ? grabber.getSampleRate() : 44_100);
            recorder.setAudioChannels(Math.max(grabber.getAudioChannels(), 1));
            recorder.start();
            Frame frame;
            while ((frame = grabber.grabSamples()) != null) {
                recorder.record(frame);
            }
        }
    } catch (Exception e) {
        throw new UncheckedIOException("silenceRemoveMp3 failed: " + input, new IOException(e.getMessage(), e));
    }
}
```

Commit whichever approach works — do not ship both. The grabber approach is safer because `setOption` feeds the raw libavformat option map before `avformat_open_input`, which JavaCV does honour for filter graphs. Resolve this before starting Task 8.

- [ ] **Step 4: Run test again**

```bash
./mvnw test -Dtest=FFmpegSilenceRemoveTest
```

Expected: PASS (missing/empty input tests pass; actual audio processing is tested end-to-end only in Plan C)

- [ ] **Step 5: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/ffmpeg/FFmpegOperations.java \
        src/test/java/kg/automation/rest/automatation/ffmpeg/FFmpegSilenceRemoveTest.java
git commit -m "feat: FFmpegOperations.silenceRemoveMp3 — trim leading/trailing silence"
```

---

## Task 5: SilenceServiceClient (HTTP client for Demucs)

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/gateway/SilenceServiceClient.java`
- Create: `src/test/java/kg/automation/rest/automatation/gateway/SilenceServiceClientTest.java`

- [ ] **Step 1: Write failing test**

```java
// src/test/java/kg/automation/rest/automatation/gateway/SilenceServiceClientTest.java
package kg.automation.rest.automatation.gateway;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.test.web.client.MockRestServiceServer;
import org.springframework.web.client.RestTemplate;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;

class SilenceServiceClientTest {

    private final RestTemplate restTemplate = new RestTemplate();
    private final MockRestServiceServer mockServer = MockRestServiceServer.createServer(restTemplate);
    private final SilenceServiceClient client = new SilenceServiceClient(restTemplate, "http://localhost:8000");

    @Test
    void denoise_returnsOutputPath_onSuccess(@TempDir Path tmp) throws IOException {
        Path input = tmp.resolve("chapter1.mp3");
        Files.writeString(input, "fake-mp3");
        Path outputDir = tmp.resolve("out");
        Files.createDirectories(outputDir);

        mockServer.expect(requestTo("http://localhost:8000/process"))
                  .andExpect(method(HttpMethod.POST))
                  .andRespond(withSuccess(
                          "{\"status\":\"success\",\"output_path\":\"/tmp/processed/chapter1_demucs.mp3\"}",
                          MediaType.APPLICATION_JSON));

        String result = client.denoise(input, outputDir);
        assertThat(result).isNotBlank();
        mockServer.verify();
    }

    @Test
    void denoise_throwsOnMissingInput(@TempDir Path tmp) {
        Path missing = tmp.resolve("missing.mp3");
        Path outputDir = tmp;
        assertThatThrownBy(() -> client.denoise(missing, outputDir))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
./mvnw test -Dtest=SilenceServiceClientTest
```

Expected: FAIL — `SilenceServiceClient not found`

- [ ] **Step 3: Create SilenceServiceClient**

```java
// src/main/java/kg/automation/rest/automatation/gateway/SilenceServiceClient.java
package kg.automation.rest.automatation.gateway;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.FileSystemResource;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestTemplate;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

/**
 * HTTP client for the Demucs noise-removal microservice at {@code silence.service.url} (:8000).
 * Calls {@code POST /process} with the audio file as multipart form-data.
 * Blocks until Demucs returns — can take several minutes for long audio.
 */
@Slf4j
@Service
public class SilenceServiceClient {

    private final RestTemplate restTemplate;
    private final String baseUrl;

    public SilenceServiceClient(
            RestTemplate restTemplate,
            @Value("${silence.service.url:http://localhost:8000}") String baseUrl) {
        this.restTemplate = restTemplate;
        this.baseUrl = baseUrl;
    }

    /**
     * Sends {@code inputFile} to Demucs. The service writes the denoised file and returns its path.
     *
     * @param inputFile  existing MP3 file to denoise
     * @param outputDir  directory where the caller expects output (used for logging only; actual
     *                   output path comes from the service response)
     * @return absolute path of the denoised file as reported by the service
     */
    public String denoise(Path inputFile, Path outputDir) {
        if (!Files.exists(inputFile)) {
            throw new IllegalArgumentException("Input file does not exist: " + inputFile);
        }
        log.info("Sending {} to Demucs noise-removal (outputDir hint: {})", inputFile, outputDir);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.MULTIPART_FORM_DATA);

        MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
        body.add("file", new FileSystemResource(inputFile));

        @SuppressWarnings("unchecked")
        Map<String, Object> response = restTemplate.postForObject(
                baseUrl + "/process",
                new HttpEntity<>(body, headers),
                Map.class);

        if (response == null || !"success".equals(response.get("status"))) {
            String error = response != null ? String.valueOf(response.get("error")) : "null response";
            throw new IllegalStateException("Demucs returned failure for " + inputFile + ": " + error);
        }
        String outputPath = String.valueOf(response.get("output_path"));
        log.info("Demucs completed: {} → {}", inputFile, outputPath);
        return outputPath;
    }
}
```

- [ ] **Step 4: Run test**

```bash
./mvnw test -Dtest=SilenceServiceClientTest
```

Expected: PASS

- [ ] **Step 5: Add @MockitoBean to AutomatationApplicationTests**

`SilenceServiceClient` is now a Spring bean. `contextLoads()` will fail without a mock.
Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
SilenceServiceClient silenceServiceClient;
```

And the import:
```java
import kg.automation.rest.automatation.gateway.SilenceServiceClient;
```

Run to verify context still loads:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/gateway/SilenceServiceClient.java \
        src/test/java/kg/automation/rest/automatation/gateway/SilenceServiceClientTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: SilenceServiceClient — HTTP client for Demucs noise-removal microservice"
```

---

## Task 6: WhisperServiceClient (HTTP client for transcription)

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/gateway/WhisperServiceClient.java`
- Create: `src/test/java/kg/automation/rest/automatation/gateway/WhisperServiceClientTest.java`

- [ ] **Step 1: Write failing test**

```java
// src/test/java/kg/automation/rest/automatation/gateway/WhisperServiceClientTest.java
package kg.automation.rest.automatation.gateway;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.test.web.client.MockRestServiceServer;
import org.springframework.web.client.RestTemplate;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;

class WhisperServiceClientTest {

    private final RestTemplate restTemplate = new RestTemplate();
    private final MockRestServiceServer mockServer = MockRestServiceServer.createServer(restTemplate);
    private final WhisperServiceClient client = new WhisperServiceClient(restTemplate, "http://localhost:8002");

    @Test
    void transcribe_returnsText_onSuccess(@TempDir Path tmp) throws IOException {
        Path input = tmp.resolve("chapter1.mp3");
        Files.writeString(input, "fake-mp3");

        mockServer.expect(requestTo("http://localhost:8002/transcribe"))
                  .andExpect(method(HttpMethod.POST))
                  .andRespond(withSuccess(
                          "{\"text\":\"Chapter one. Once upon a time.\",\"segments\":[]}",
                          MediaType.APPLICATION_JSON));

        String text = client.transcribe(input);
        assertThat(text).contains("Chapter one");
        mockServer.verify();
    }

    @Test
    void transcribe_throwsOnMissingInput(@TempDir Path tmp) {
        Path missing = tmp.resolve("missing.mp3");
        assertThatThrownBy(() -> client.transcribe(missing))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run to verify failure**

```bash
./mvnw test -Dtest=WhisperServiceClientTest
```

Expected: FAIL — `WhisperServiceClient not found`

- [ ] **Step 3: Create WhisperServiceClient**

```java
// src/main/java/kg/automation/rest/automatation/gateway/WhisperServiceClient.java
package kg.automation.rest.automatation.gateway;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.FileSystemResource;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestTemplate;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

/**
 * HTTP client for the Whisper transcription microservice at {@code whisper.service.url} (:8002).
 * Calls {@code POST /transcribe} with the audio file as multipart form-data.
 * Blocks until Whisper returns — can take several minutes for long audio.
 */
@Slf4j
@Service
public class WhisperServiceClient {

    private final RestTemplate restTemplate;
    private final String baseUrl;

    public WhisperServiceClient(
            RestTemplate restTemplate,
            @Value("${whisper.service.url:http://localhost:8002}") String baseUrl) {
        this.restTemplate = restTemplate;
        this.baseUrl = baseUrl;
    }

    /**
     * Transcribes {@code audioFile} using Whisper.
     *
     * @return the full transcription text
     */
    public String transcribe(Path audioFile) {
        if (!Files.exists(audioFile)) {
            throw new IllegalArgumentException("Audio file does not exist: " + audioFile);
        }
        log.info("Sending {} to Whisper for transcription", audioFile);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.MULTIPART_FORM_DATA);
        MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
        body.add("file", new FileSystemResource(audioFile));

        @SuppressWarnings("unchecked")
        Map<String, Object> response = restTemplate.postForObject(
                baseUrl + "/transcribe",
                new HttpEntity<>(body, headers),
                Map.class);

        if (response == null) {
            throw new IllegalStateException("Whisper returned null response for: " + audioFile);
        }
        String text = String.valueOf(response.getOrDefault("text", ""));
        log.info("Whisper transcription complete for {}: {} chars", audioFile, text.length());
        return text;
    }
}
```

- [ ] **Step 4: Run test**

```bash
./mvnw test -Dtest=WhisperServiceClientTest
```

Expected: PASS

- [ ] **Step 5: Add @MockitoBean to AutomatationApplicationTests**

Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
WhisperServiceClient whisperServiceClient;
```

And the import:
```java
import kg.automation.rest.automatation.gateway.WhisperServiceClient;
```

Run:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/gateway/WhisperServiceClient.java \
        src/test/java/kg/automation/rest/automatation/gateway/WhisperServiceClientTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: WhisperServiceClient — HTTP client for Whisper transcription microservice"
```

---

## Task 7: Dedicated noise-remove queue in RabbitMQ config

**Files:**
- Modify: `src/main/java/kg/automation/rest/automatation/infra/rabbit/InfraRabbitConfiguration.java`

- [ ] **Step 1: Add queue constant and beans**

Add after the existing `QUEUE_PROCESSING_LOW` constant:

```java
/** Dedicated queue for ML-heavy AUDIO_NOISE_REMOVE jobs. prefetchCount=1 protects GPU memory. */
public static final String QUEUE_AUDIO_NOISE_REMOVE = "audio.noise-remove";
public static final String QUEUE_AUDIO_NOISE_REMOVE_DLQ = "audio.noise-remove.dlq";

private static String dlkNoiseRemove() {
    return "dlq.audio.noise-remove";
}
```

Add these beans after the existing `processingLowBinding` bean:

```java
@Bean
public Queue audioNoiseRemoveDlq() {
    return QueueBuilder.durable(QUEUE_AUDIO_NOISE_REMOVE_DLQ).build();
}

@Bean
public Binding audioNoiseRemoveDlqBinding(Queue audioNoiseRemoveDlq, DirectExchange pipelineDlxExchange) {
    return BindingBuilder.bind(audioNoiseRemoveDlq).to(pipelineDlxExchange).with(dlkNoiseRemove());
}

@Bean
public Queue audioNoiseRemoveQueue() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-dead-letter-exchange", PIPELINE_DLX_EXCHANGE);
    args.put("x-dead-letter-routing-key", dlkNoiseRemove());
    return QueueBuilder.durable(QUEUE_AUDIO_NOISE_REMOVE).withArguments(args).build();
}

@Bean
public Binding audioNoiseRemoveBinding(Queue audioNoiseRemoveQueue, TopicExchange pipelineEventsExchange) {
    return BindingBuilder.bind(audioNoiseRemoveQueue).to(pipelineEventsExchange)
            .with("processing.high.audio-noise-remove");
}

@Bean
public SimpleRabbitListenerContainerFactory noiseRemoveListenerContainerFactory(
        ConnectionFactory rabbitConnectionFactory, Jackson2JsonMessageConverter pipelineRabbitMessageConverter) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(rabbitConnectionFactory);
    factory.setMessageConverter(pipelineRabbitMessageConverter);
    factory.setConcurrentConsumers(1);
    factory.setPrefetchCount(1);   // process one noise-remove job at a time — protects GPU
    return factory;
}
```

Update `allProcessingQueues()` to include the noise-remove queue:

```java
public static String[] allProcessingQueues() {
    return new String[] {
        QUEUE_PROCESSING_HIGH, QUEUE_PROCESSING_NORMAL, QUEUE_PROCESSING_LOW,
        QUEUE_AUDIO_NOISE_REMOVE
    };
}
```

- [ ] **Step 2: Run context-loads test**

```bash
./mvnw test -Dtest=AutomatationApplicationTests
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/infra/rabbit/InfraRabbitConfiguration.java
git commit -m "feat: dedicated audio.noise-remove queue with prefetchCount=1 for GPU protection"
```

---

## Task 8: AudioNoiseRemoveJobRunner

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/jobs/service/AudioNoiseRemoveJobRunner.java`
- Create: `src/test/java/kg/automation/rest/automatation/jobs/service/AudioNoiseRemoveJobRunnerTest.java`

- [ ] **Step 1: Write failing test**

```java
// src/test/java/kg/automation/rest/automatation/jobs/service/AudioNoiseRemoveJobRunnerTest.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.gateway.SilenceServiceClient;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.service.AudioStagePathService;
import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class AudioNoiseRemoveJobRunnerTest {

    @TempDir
    Path tempDir;

    ProcessingJobRepository jobRepo = mock(ProcessingJobRepository.class);
    AudioPartRepository partRepo = mock(AudioPartRepository.class);
    AudioStagePathService stagePathService = mock(AudioStagePathService.class);
    SilenceServiceClient silenceClient = mock(SilenceServiceClient.class);
    CycleFileLayout layout;
    AudioNoiseRemoveJobRunner runner;

    @BeforeEach
    void setUp() {
        layout = new CycleFileLayout(tempDir.toString());
        runner = new AudioNoiseRemoveJobRunner(jobRepo, partRepo, stagePathService, silenceClient, layout);
    }

    @Test
    void run_success_writesNoiseRemovedStageAndCompletesJob() throws Exception {
        // Arrange
        UUID jobId = UUID.randomUUID();
        long cycleId = 1L;
        long audioPartId = 10L;

        ProcessingJobEntity job = new ProcessingJobEntity();
        job.setId(jobId);
        job.setJobType(JobType.AUDIO_NOISE_REMOVE);
        job.setStatus(JobStatus.PENDING);
        job.setCycleId(cycleId);
        job.setAudioPartId(audioPartId);
        job.setCreatedAt(Instant.now());
        job.setUpdatedAt(Instant.now());
        job.setRetryCount(0);

        AudioPartEntity part = new AudioPartEntity();
        part.setId(audioPartId);
        part.setCycleId(cycleId);
        part.setPartNumber(1);
        part.setProcessingStatus(PartProcessingStatus.PENDING);
        part.setCreatedAt(Instant.now());
        part.setUpdatedAt(Instant.now());

        // Create a real source MP3 file so the runner finds it
        Path sourceDir = layout.partSourceDir(cycleId, audioPartId);
        Path sourceFile = sourceDir.resolve("track01.mp3");
        Files.writeString(sourceFile, "fake-mp3-content");

        // Demucs returns a path — the file must actually exist on disk for the runner to accept it
        Path noiseRemovedDir = layout.partNoiseRemovedDir(cycleId, audioPartId);
        Path denoisedFile = noiseRemovedDir.resolve("track01_denoised.mp3");
        Files.writeString(denoisedFile, "fake-denoised-content");  // runner validates path exists
        String denoisedPath = denoisedFile.toString();

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(audioPartId)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(silenceClient.denoise(eq(sourceFile), any())).thenReturn(denoisedPath);

        // Act
        runner.run(jobId);

        // Assert
        assertThat(job.getStatus()).isEqualTo(JobStatus.COMPLETED);
        assertThat(job.getStartedAt()).isNotNull();
        assertThat(job.getCompletedAt()).isNotNull();
        verify(stagePathService).putPath(eq(audioPartId), eq(AudioStageType.NOISE_REMOVED), eq(denoisedPath));
    }

    @Test
    void run_silenceClientThrows_marksJobFailed() throws Exception {
        UUID jobId = UUID.randomUUID();
        long cycleId = 2L;
        long audioPartId = 20L;

        ProcessingJobEntity job = new ProcessingJobEntity();
        job.setId(jobId);
        job.setJobType(JobType.AUDIO_NOISE_REMOVE);
        job.setStatus(JobStatus.PENDING);
        job.setCycleId(cycleId);
        job.setAudioPartId(audioPartId);
        job.setCreatedAt(Instant.now());
        job.setUpdatedAt(Instant.now());
        job.setRetryCount(0);

        AudioPartEntity part = new AudioPartEntity();
        part.setId(audioPartId);
        part.setCycleId(cycleId);
        part.setPartNumber(1);
        part.setProcessingStatus(PartProcessingStatus.PENDING);
        part.setCreatedAt(Instant.now());
        part.setUpdatedAt(Instant.now());

        Path sourceDir = layout.partSourceDir(cycleId, audioPartId);
        Files.writeString(sourceDir.resolve("track01.mp3"), "fake");

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(audioPartId)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(silenceClient.denoise(any(), any())).thenThrow(new RuntimeException("Demucs unreachable"));

        runner.run(jobId);

        assertThat(job.getStatus()).isEqualTo(JobStatus.FAILED);
        assertThat(job.getErrorMessage()).contains("Demucs unreachable");
        assertThat(part.getProcessingStatus()).isEqualTo(PartProcessingStatus.FAILED);
    }

    @Test
    void run_denoiseReturnsNonExistentPath_marksJobFailed() throws Exception {
        UUID jobId = UUID.randomUUID();
        long cycleId = 3L;
        long audioPartId = 30L;

        ProcessingJobEntity job = new ProcessingJobEntity();
        job.setId(jobId);
        job.setJobType(JobType.AUDIO_NOISE_REMOVE);
        job.setStatus(JobStatus.PENDING);
        job.setCycleId(cycleId);
        job.setAudioPartId(audioPartId);
        job.setCreatedAt(Instant.now());
        job.setUpdatedAt(Instant.now());
        job.setRetryCount(0);

        AudioPartEntity part = new AudioPartEntity();
        part.setId(audioPartId);
        part.setCycleId(cycleId);
        part.setPartNumber(1);
        part.setProcessingStatus(PartProcessingStatus.PENDING);
        part.setCreatedAt(Instant.now());
        part.setUpdatedAt(Instant.now());

        Path sourceDir = layout.partSourceDir(cycleId, audioPartId);
        Files.writeString(sourceDir.resolve("track01.mp3"), "fake");

        // Demucs returns a path that does not exist on this machine
        String phantomPath = "/nonexistent/demucs/output/track01.mp3";

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(audioPartId)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(silenceClient.denoise(any(), any())).thenReturn(phantomPath);

        runner.run(jobId);

        assertThat(job.getStatus()).isEqualTo(JobStatus.FAILED);
        assertThat(job.getErrorMessage()).containsIgnoringCase("not found");
    }
}
```

- [ ] **Step 2: Run to verify failure**

```bash
./mvnw test -Dtest=AudioNoiseRemoveJobRunnerTest
```

Expected: FAIL — `AudioNoiseRemoveJobRunner not found`, also `partNoiseRemovedDir not found on CycleFileLayout`

- [ ] **Step 3: Add partNoiseRemovedDir to CycleFileLayout**

Add after `partCompressedDir` in `CycleFileLayout.java`:

```java
public Path partNoiseRemovedDir(long cycleId, long partId) {
    return mkdir(cycleDir(cycleId).resolve("part_" + partId).resolve("noise_removed"));
}

public Path partTrimmedDir(long cycleId, long partId) {
    return mkdir(cycleDir(cycleId).resolve("part_" + partId).resolve("trimmed"));
}
```

- [ ] **Step 4: Create AudioNoiseRemoveJobRunner**

```java
// src/main/java/kg/automation/rest/automatation/jobs/service/AudioNoiseRemoveJobRunner.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.gateway.SilenceServiceClient;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.service.AudioStagePathService;
import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.stream.Stream;

/**
 * Handles AUDIO_NOISE_REMOVE jobs.
 * Reads all MP3s from {@link CycleFileLayout#partSourceDir}, sends each to Demucs,
 * writes denoised paths to {@code audio_stage.NOISE_REMOVED}.
 * If multiple source files exist (multi-track), all are denoised; the last output path is stored.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AudioNoiseRemoveJobRunner {

    private final ProcessingJobRepository jobRepository;
    private final AudioPartRepository audioPartRepository;
    private final AudioStagePathService audioStagePathService;
    private final SilenceServiceClient silenceServiceClient;
    private final CycleFileLayout cycleFileLayout;

    @Transactional
    public void run(UUID jobId) {
        ProcessingJobEntity job = jobRepository.findById(jobId).orElseThrow(
                () -> new IllegalStateException("job_queue row not found: " + jobId));

        if (job.getStatus() != JobStatus.PENDING) {
            log.warn("AUDIO_NOISE_REMOVE job {} is not PENDING ({}), skipping", jobId, job.getStatus());
            return;
        }

        job.setStatus(JobStatus.IN_PROGRESS);
        job.setStartedAt(Instant.now());
        jobRepository.save(job);

        long audioPartId = job.getAudioPartId();
        AudioPartEntity part = audioPartRepository.findById(audioPartId).orElseThrow(
                () -> new IllegalStateException("audio_part not found: " + audioPartId));
        part.setProcessingStatus(PartProcessingStatus.IN_PROGRESS);
        audioPartRepository.save(part);

        String lastDenoisedPath = null;
        try {
            Path sourceDir = cycleFileLayout.partSourceDir(job.getCycleId(), audioPartId);
            Path outputDir = cycleFileLayout.partNoiseRemovedDir(job.getCycleId(), audioPartId);
            List<Path> sources = sortedMp3s(sourceDir);

            if (sources.isEmpty()) {
                throw new IllegalStateException("No MP3 files found in source dir: " + sourceDir);
            }

            for (Path source : sources) {
                String returnedPath = silenceServiceClient.denoise(source, outputDir);
                // Demucs owns its output path — validate it actually landed on disk before trusting it
                if (!Files.exists(Path.of(returnedPath))) {
                    throw new IllegalStateException(
                            "Demucs returned output path not found on disk: " + returnedPath
                            + " (source: " + source + ")");
                }
                lastDenoisedPath = returnedPath;
                log.debug("Denoised {} → {}", source, lastDenoisedPath);
            }

            audioStagePathService.putPath(audioPartId, AudioStageType.NOISE_REMOVED, lastDenoisedPath);
            part.setProcessingStatus(PartProcessingStatus.IN_PROGRESS); // still in progress — more stages ahead
            audioPartRepository.save(part);

            job.setStatus(JobStatus.COMPLETED);
            job.setResultPayload("{\"denoisedPath\":\"" + lastDenoisedPath + "\"}");
        } catch (Exception ex) {
            log.error("AUDIO_NOISE_REMOVE job {} failed: {}", jobId, ex.getMessage(), ex);
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(ex.getMessage());
            part.setProcessingStatus(PartProcessingStatus.FAILED);
            audioPartRepository.save(part);
        }

        job.setCompletedAt(Instant.now());
        jobRepository.save(job);
    }

    private static List<Path> sortedMp3s(Path dir) throws IOException {
        try (Stream<Path> stream = Files.list(dir)) {
            return stream
                    .filter(p -> p.getFileName().toString().toLowerCase().endsWith(".mp3"))
                    .sorted()
                    .toList();
        }
    }
}
```

- [ ] **Step 5: Run test**

```bash
./mvnw test -Dtest=AudioNoiseRemoveJobRunnerTest
```

Expected: PASS

- [ ] **Step 6: Add @MockitoBean to AutomatationApplicationTests**

`AudioNoiseRemoveJobRunner` is a new Spring bean. Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
AudioNoiseRemoveJobRunner audioNoiseRemoveJobRunner;
```

And the import:
```java
import kg.automation.rest.automatation.jobs.service.AudioNoiseRemoveJobRunner;
```

Run:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/pipeline/service/CycleFileLayout.java \
        src/main/java/kg/automation/rest/automatation/jobs/service/AudioNoiseRemoveJobRunner.java \
        src/test/java/kg/automation/rest/automatation/jobs/service/AudioNoiseRemoveJobRunnerTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: AudioNoiseRemoveJobRunner — AUDIO_NOISE_REMOVE via Demucs HTTP client"
```

---

## Task 9: AudioTrimJobRunner

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/jobs/service/AudioTrimJobRunner.java`
- Create: `src/test/java/kg/automation/rest/automatation/jobs/service/AudioTrimJobRunnerTest.java`

- [ ] **Step 1: Write failing test**

```java
// src/test/java/kg/automation/rest/automatation/jobs/service/AudioTrimJobRunnerTest.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.ffmpeg.FFmpegOperations;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import kg.automation.rest.automatation.pipeline.service.AudioStagePathService;
import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class AudioTrimJobRunnerTest {

    @TempDir
    Path tempDir;

    ProcessingJobRepository jobRepo = mock(ProcessingJobRepository.class);
    AudioPartRepository partRepo = mock(AudioPartRepository.class);
    AudioStageRepository stageRepo = mock(AudioStageRepository.class);
    AudioStagePathService stagePathService = mock(AudioStagePathService.class);
    FFmpegOperations ffmpeg = mock(FFmpegOperations.class);
    CycleFileLayout layout;
    AudioTrimJobRunner runner;

    @BeforeEach
    void setUp() {
        layout = new CycleFileLayout(tempDir.toString());
        runner = new AudioTrimJobRunner(jobRepo, partRepo, stageRepo, stagePathService, ffmpeg, layout);
    }

    @Test
    void run_success_writesTrimedStage() throws Exception {
        UUID jobId = UUID.randomUUID();
        long cycleId = 1L;
        long audioPartId = 10L;

        ProcessingJobEntity job = buildPendingJob(jobId, cycleId, audioPartId);

        AudioPartEntity part = buildPart(audioPartId, cycleId);

        // noise-removed stage exists
        Path noiseRemovedDir = layout.partNoiseRemovedDir(cycleId, audioPartId);
        Path noiseRemovedFile = noiseRemovedDir.resolve("chapter1_denoised.mp3");
        Files.writeString(noiseRemovedFile, "fake");

        AudioStageEntity noiseStage = new AudioStageEntity();
        noiseStage.setAudioPartId(audioPartId);
        noiseStage.setStage(AudioStageType.NOISE_REMOVED);
        noiseStage.setFilePath(noiseRemovedFile.toString());

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(audioPartId)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(stageRepo.findByAudioPartIdAndStage(audioPartId, AudioStageType.NOISE_REMOVED))
                .thenReturn(Optional.of(noiseStage));
        doNothing().when(ffmpeg).silenceRemoveMp3(any(), any());

        runner.run(jobId);

        assertThat(job.getStatus()).isEqualTo(JobStatus.COMPLETED);
        verify(stagePathService).putPath(eq(audioPartId), eq(AudioStageType.TRIMMED), anyString());
    }

    @Test
    void run_noiseRemovedStageMissing_marksJobFailed() {
        UUID jobId = UUID.randomUUID();
        ProcessingJobEntity job = buildPendingJob(jobId, 1L, 10L);
        AudioPartEntity part = buildPart(10L, 1L);

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(10L)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(stageRepo.findByAudioPartIdAndStage(10L, AudioStageType.NOISE_REMOVED))
                .thenReturn(Optional.empty());

        runner.run(jobId);

        assertThat(job.getStatus()).isEqualTo(JobStatus.FAILED);
        assertThat(job.getErrorMessage()).contains("NOISE_REMOVED");
    }

    private static ProcessingJobEntity buildPendingJob(UUID id, long cycleId, long audioPartId) {
        ProcessingJobEntity j = new ProcessingJobEntity();
        j.setId(id);
        j.setJobType(JobType.AUDIO_TRIM);
        j.setStatus(JobStatus.PENDING);
        j.setCycleId(cycleId);
        j.setAudioPartId(audioPartId);
        j.setCreatedAt(Instant.now());
        j.setUpdatedAt(Instant.now());
        j.setRetryCount(0);
        return j;
    }

    private static AudioPartEntity buildPart(long id, long cycleId) {
        AudioPartEntity p = new AudioPartEntity();
        p.setId(id);
        p.setCycleId(cycleId);
        p.setPartNumber(1);
        p.setProcessingStatus(PartProcessingStatus.PENDING);
        p.setCreatedAt(Instant.now());
        p.setUpdatedAt(Instant.now());
        return p;
    }
}
```

- [ ] **Step 2: Run to verify failure**

```bash
./mvnw test -Dtest=AudioTrimJobRunnerTest
```

Expected: FAIL — `AudioTrimJobRunner not found`

- [ ] **Step 3: Create AudioTrimJobRunner**

```java
// src/main/java/kg/automation/rest/automatation/jobs/service/AudioTrimJobRunner.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.ffmpeg.FFmpegOperations;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import kg.automation.rest.automatation.pipeline.service.AudioStagePathService;
import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.nio.file.Path;
import java.time.Instant;
import java.util.UUID;

/**
 * Handles AUDIO_TRIM jobs.
 * Reads the NOISE_REMOVED stage file, applies FFmpeg silence-trim, writes TRIMMED stage.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AudioTrimJobRunner {

    private final ProcessingJobRepository jobRepository;
    private final AudioPartRepository audioPartRepository;
    private final AudioStageRepository audioStageRepository;
    private final AudioStagePathService audioStagePathService;
    private final FFmpegOperations ffmpegOperations;
    private final CycleFileLayout cycleFileLayout;

    @Transactional
    public void run(UUID jobId) {
        ProcessingJobEntity job = jobRepository.findById(jobId).orElseThrow(
                () -> new IllegalStateException("job_queue row not found: " + jobId));

        if (job.getStatus() != JobStatus.PENDING) {
            log.warn("AUDIO_TRIM job {} is not PENDING ({}), skipping", jobId, job.getStatus());
            return;
        }

        job.setStatus(JobStatus.IN_PROGRESS);
        job.setStartedAt(Instant.now());
        jobRepository.save(job);

        long audioPartId = job.getAudioPartId();
        AudioPartEntity part = audioPartRepository.findById(audioPartId).orElseThrow(
                () -> new IllegalStateException("audio_part not found: " + audioPartId));
        part.setProcessingStatus(PartProcessingStatus.IN_PROGRESS);
        audioPartRepository.save(part);

        try {
            AudioStageEntity noiseRemovedStage = audioStageRepository
                    .findByAudioPartIdAndStage(audioPartId, AudioStageType.NOISE_REMOVED)
                    .orElseThrow(() -> new IllegalStateException(
                            "NOISE_REMOVED stage not found for audio_part " + audioPartId
                            + " — AUDIO_TRIM requires NOISE_REMOVED to complete first"));

            Path input = Path.of(noiseRemovedStage.getFilePath());
            Path outputDir = cycleFileLayout.partTrimmedDir(job.getCycleId(), audioPartId);
            Path output = outputDir.resolve(input.getFileName().toString().replace(".mp3", "_trimmed.mp3"));

            ffmpegOperations.silenceRemoveMp3(input, output);

            audioStagePathService.putPath(audioPartId, AudioStageType.TRIMMED, output.toString());
            job.setStatus(JobStatus.COMPLETED);
            job.setResultPayload("{\"trimmedPath\":\"" + output + "\"}");
        } catch (Exception ex) {
            log.error("AUDIO_TRIM job {} failed: {}", jobId, ex.getMessage(), ex);
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(ex.getMessage());
            part.setProcessingStatus(PartProcessingStatus.FAILED);
            audioPartRepository.save(part);
        }

        job.setCompletedAt(Instant.now());
        jobRepository.save(job);
    }
}
```

- [ ] **Step 4: Run test**

```bash
./mvnw test -Dtest=AudioTrimJobRunnerTest
```

Expected: PASS

- [ ] **Step 5: Add @MockitoBean to AutomatationApplicationTests**

Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
AudioTrimJobRunner audioTrimJobRunner;
```

And the import:
```java
import kg.automation.rest.automatation.jobs.service.AudioTrimJobRunner;
```

Run:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/service/AudioTrimJobRunner.java \
        src/test/java/kg/automation/rest/automatation/jobs/service/AudioTrimJobRunnerTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: AudioTrimJobRunner — AUDIO_TRIM silence removal via FFmpegOperations"
```

---

## Task 10: AudioCompressJobRunner

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/jobs/service/AudioCompressJobRunner.java`
- Create: `src/test/java/kg/automation/rest/automatation/jobs/service/AudioCompressJobRunnerTest.java`

- [ ] **Step 1: Write failing test**

```java
// src/test/java/kg/automation/rest/automatation/jobs/service/AudioCompressJobRunnerTest.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.ffmpeg.FFmpegOperations;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import kg.automation.rest.automatation.pipeline.service.AudioStagePathService;
import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class AudioCompressJobRunnerTest {

    @TempDir
    Path tempDir;

    ProcessingJobRepository jobRepo = mock(ProcessingJobRepository.class);
    AudioPartRepository partRepo = mock(AudioPartRepository.class);
    AudioStageRepository stageRepo = mock(AudioStageRepository.class);
    AudioStagePathService stagePathService = mock(AudioStagePathService.class);
    FFmpegOperations ffmpeg = mock(FFmpegOperations.class);
    CycleFileLayout layout;
    AudioCompressJobRunner runner;

    @BeforeEach
    void setUp() {
        layout = new CycleFileLayout(tempDir.toString());
        runner = new AudioCompressJobRunner(jobRepo, partRepo, stageRepo, stagePathService, ffmpeg, layout);
    }

    @Test
    void run_success_writesCompressedStageAndCompletesJob() throws Exception {
        UUID jobId = UUID.randomUUID();
        long cycleId = 1L;
        long audioPartId = 10L;

        ProcessingJobEntity job = new ProcessingJobEntity();
        job.setId(jobId);
        job.setJobType(JobType.AUDIO_COMPRESS);
        job.setStatus(JobStatus.PENDING);
        job.setCycleId(cycleId);
        job.setAudioPartId(audioPartId);
        job.setCreatedAt(Instant.now());
        job.setUpdatedAt(Instant.now());
        job.setRetryCount(0);

        AudioPartEntity part = new AudioPartEntity();
        part.setId(audioPartId);
        part.setCycleId(cycleId);
        part.setPartNumber(1);
        part.setProcessingStatus(PartProcessingStatus.IN_PROGRESS);
        part.setCreatedAt(Instant.now());
        part.setUpdatedAt(Instant.now());

        // CONCATENATED stage exists
        Path concatFile = layout.partConcatFile(cycleId, audioPartId);
        Files.createDirectories(concatFile.getParent());
        Files.writeString(concatFile, "fake-concat-mp3");

        AudioStageEntity concatStage = new AudioStageEntity();
        concatStage.setAudioPartId(audioPartId);
        concatStage.setStage(AudioStageType.CONCATENATED);
        concatStage.setFilePath(concatFile.toString());

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(audioPartId)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(stageRepo.findByAudioPartIdAndStage(audioPartId, AudioStageType.CONCATENATED))
                .thenReturn(Optional.of(concatStage));
        doNothing().when(ffmpeg).transcodeMp3Lame240k44k1(any(), any());

        runner.run(jobId);

        assertThat(job.getStatus()).isEqualTo(JobStatus.COMPLETED);
        verify(stagePathService).putPath(eq(audioPartId), eq(AudioStageType.COMPRESSED), anyString());
        verify(ffmpeg).transcodeMp3Lame240k44k1(eq(concatFile), any());
    }
}
```

- [ ] **Step 2: Run to verify failure**

```bash
./mvnw test -Dtest=AudioCompressJobRunnerTest
```

Expected: FAIL — `AudioCompressJobRunner not found`

- [ ] **Step 3: Create AudioCompressJobRunner**

```java
// src/main/java/kg/automation/rest/automatation/jobs/service/AudioCompressJobRunner.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.ffmpeg.FFmpegOperations;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import kg.automation.rest.automatation.pipeline.service.AudioStagePathService;
import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.nio.file.Path;
import java.time.Instant;
import java.util.UUID;

/**
 * Handles AUDIO_COMPRESS jobs.
 * Reads the CONCATENATED stage file, re-encodes to 240kbps, writes COMPRESSED stage.
 * On success, marks audio_part as COMPLETED (audio pipeline done for this part).
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AudioCompressJobRunner {

    private final ProcessingJobRepository jobRepository;
    private final AudioPartRepository audioPartRepository;
    private final AudioStageRepository audioStageRepository;
    private final AudioStagePathService audioStagePathService;
    private final FFmpegOperations ffmpegOperations;
    private final CycleFileLayout cycleFileLayout;

    @Transactional
    public void run(UUID jobId) {
        ProcessingJobEntity job = jobRepository.findById(jobId).orElseThrow(
                () -> new IllegalStateException("job_queue row not found: " + jobId));

        if (job.getStatus() != JobStatus.PENDING) {
            log.warn("AUDIO_COMPRESS job {} is not PENDING ({}), skipping", jobId, job.getStatus());
            return;
        }

        job.setStatus(JobStatus.IN_PROGRESS);
        job.setStartedAt(Instant.now());
        jobRepository.save(job);

        long audioPartId = job.getAudioPartId();
        AudioPartEntity part = audioPartRepository.findById(audioPartId).orElseThrow(
                () -> new IllegalStateException("audio_part not found: " + audioPartId));

        try {
            AudioStageEntity concatStage = audioStageRepository
                    .findByAudioPartIdAndStage(audioPartId, AudioStageType.CONCATENATED)
                    .orElseThrow(() -> new IllegalStateException(
                            "CONCATENATED stage not found for audio_part " + audioPartId));

            Path input = Path.of(concatStage.getFilePath());
            Path outputDir = cycleFileLayout.partCompressedDir(job.getCycleId(), audioPartId);
            Path output = outputDir.resolve(input.getFileName().toString().replace(".mp3", "_compressed.mp3"));

            ffmpegOperations.transcodeMp3Lame240k44k1(input, output);

            audioStagePathService.putPath(audioPartId, AudioStageType.COMPRESSED, output.toString());
            // Audio pipeline complete for this part
            part.setProcessingStatus(PartProcessingStatus.COMPLETED);
            audioPartRepository.save(part);

            job.setStatus(JobStatus.COMPLETED);
            job.setResultPayload("{\"compressedPath\":\"" + output + "\"}");
        } catch (Exception ex) {
            log.error("AUDIO_COMPRESS job {} failed: {}", jobId, ex.getMessage(), ex);
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(ex.getMessage());
            part.setProcessingStatus(PartProcessingStatus.FAILED);
            audioPartRepository.save(part);
        }

        job.setCompletedAt(Instant.now());
        jobRepository.save(job);
    }
}
```

- [ ] **Step 4: Run test**

```bash
./mvnw test -Dtest=AudioCompressJobRunnerTest
```

Expected: PASS

- [ ] **Step 5: Add @MockitoBean to AutomatationApplicationTests**

Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
AudioCompressJobRunner audioCompressJobRunner;
```

And the import:
```java
import kg.automation.rest.automatation.jobs.service.AudioCompressJobRunner;
```

Run:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/service/AudioCompressJobRunner.java \
        src/test/java/kg/automation/rest/automatation/jobs/service/AudioCompressJobRunnerTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: AudioCompressJobRunner — AUDIO_COMPRESS via FFmpegOperations.transcodeMp3Lame240k44k1"
```

---

## Task 11: AudioDurationJobRunner

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/jobs/service/AudioDurationJobRunner.java`
- Create: `src/test/java/kg/automation/rest/automatation/jobs/service/AudioDurationJobRunnerTest.java`

- [ ] **Step 1: Write failing test**

```java
// src/test/java/kg/automation/rest/automatation/jobs/service/AudioDurationJobRunnerTest.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.ffmpeg.FFmpegOperations;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class AudioDurationJobRunnerTest {

    ProcessingJobRepository jobRepo = mock(ProcessingJobRepository.class);
    AudioPartRepository partRepo = mock(AudioPartRepository.class);
    AudioStageRepository stageRepo = mock(AudioStageRepository.class);
    FFmpegOperations ffmpeg = mock(FFmpegOperations.class);
    AudioDurationJobRunner runner;

    @BeforeEach
    void setUp() {
        runner = new AudioDurationJobRunner(jobRepo, partRepo, stageRepo, ffmpeg);
    }

    @Test
    void run_success_writesDurationSecondsAndCompletesJob() {
        UUID jobId = UUID.randomUUID();
        long audioPartId = 10L;

        ProcessingJobEntity job = new ProcessingJobEntity();
        job.setId(jobId);
        job.setJobType(JobType.AUDIO_DURATION);
        job.setStatus(JobStatus.PENDING);
        job.setCycleId(1L);
        job.setAudioPartId(audioPartId);
        job.setCreatedAt(Instant.now());
        job.setUpdatedAt(Instant.now());
        job.setRetryCount(0);

        AudioPartEntity part = new AudioPartEntity();
        part.setId(audioPartId);
        part.setCycleId(1L);
        part.setPartNumber(1);
        part.setProcessingStatus(PartProcessingStatus.COMPLETED);
        part.setCreatedAt(Instant.now());
        part.setUpdatedAt(Instant.now());

        AudioStageEntity compressedStage = new AudioStageEntity();
        compressedStage.setAudioPartId(audioPartId);
        compressedStage.setStage(AudioStageType.COMPRESSED);
        compressedStage.setFilePath("/tmp/part10_compressed.mp3");

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(audioPartId)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        // prefers COMPRESSED; falls back to CONCATENATED if absent
        when(stageRepo.findByAudioPartIdAndStage(audioPartId, AudioStageType.COMPRESSED))
                .thenReturn(Optional.of(compressedStage));
        when(ffmpeg.probeDurationSeconds(any(Path.class))).thenReturn(3600L);

        runner.run(jobId);

        assertThat(job.getStatus()).isEqualTo(JobStatus.COMPLETED);
        assertThat(part.getDurationSeconds()).isEqualTo(3600L);
        verify(partRepo).save(part);
    }

    @Test
    void run_noStageFound_marksJobFailed() {
        UUID jobId = UUID.randomUUID();
        ProcessingJobEntity job = new ProcessingJobEntity();
        job.setId(jobId);
        job.setJobType(JobType.AUDIO_DURATION);
        job.setStatus(JobStatus.PENDING);
        job.setCycleId(1L);
        job.setAudioPartId(99L);
        job.setCreatedAt(Instant.now());
        job.setUpdatedAt(Instant.now());
        job.setRetryCount(0);

        AudioPartEntity part = new AudioPartEntity();
        part.setId(99L);
        part.setCycleId(1L);
        part.setPartNumber(1);
        part.setProcessingStatus(PartProcessingStatus.IN_PROGRESS);
        part.setCreatedAt(Instant.now());
        part.setUpdatedAt(Instant.now());

        when(jobRepo.findById(jobId)).thenReturn(Optional.of(job));
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findById(99L)).thenReturn(Optional.of(part));
        when(partRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(stageRepo.findByAudioPartIdAndStage(99L, AudioStageType.COMPRESSED))
                .thenReturn(Optional.empty());
        when(stageRepo.findByAudioPartIdAndStage(99L, AudioStageType.CONCATENATED))
                .thenReturn(Optional.empty());

        runner.run(jobId);

        assertThat(job.getStatus()).isEqualTo(JobStatus.FAILED);
    }
}
```

- [ ] **Step 2: Run to verify failure**

```bash
./mvnw test -Dtest=AudioDurationJobRunnerTest
```

Expected: FAIL

- [ ] **Step 3: Create AudioDurationJobRunner**

```java
// src/main/java/kg/automation/rest/automatation/jobs/service/AudioDurationJobRunner.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.ffmpeg.FFmpegOperations;
import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.nio.file.Path;
import java.time.Instant;
import java.util.UUID;

/**
 * Handles AUDIO_DURATION jobs.
 * Probes the best available audio stage (COMPRESSED → CONCATENATED fallback)
 * and writes {@code audio_part.duration_seconds}.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AudioDurationJobRunner {

    private final ProcessingJobRepository jobRepository;
    private final AudioPartRepository audioPartRepository;
    private final AudioStageRepository audioStageRepository;
    private final FFmpegOperations ffmpegOperations;

    @Transactional
    public void run(UUID jobId) {
        ProcessingJobEntity job = jobRepository.findById(jobId).orElseThrow(
                () -> new IllegalStateException("job_queue row not found: " + jobId));

        if (job.getStatus() != JobStatus.PENDING) {
            log.warn("AUDIO_DURATION job {} is not PENDING ({}), skipping", jobId, job.getStatus());
            return;
        }

        job.setStatus(JobStatus.IN_PROGRESS);
        job.setStartedAt(Instant.now());
        jobRepository.save(job);

        long audioPartId = job.getAudioPartId();
        AudioPartEntity part = audioPartRepository.findById(audioPartId).orElseThrow(
                () -> new IllegalStateException("audio_part not found: " + audioPartId));

        try {
            // Prefer COMPRESSED; fall back to CONCATENATED
            String stagePath = audioStageRepository
                    .findByAudioPartIdAndStage(audioPartId, AudioStageType.COMPRESSED)
                    .or(() -> audioStageRepository.findByAudioPartIdAndStage(audioPartId, AudioStageType.CONCATENATED))
                    .map(s -> s.getFilePath())
                    .orElseThrow(() -> new IllegalStateException(
                            "No COMPRESSED or CONCATENATED stage found for audio_part " + audioPartId));

            Long durationSeconds = ffmpegOperations.probeDurationSeconds(Path.of(stagePath));
            if (durationSeconds == null || durationSeconds <= 0) {
                throw new IllegalStateException("probeDurationSeconds returned invalid value: " + durationSeconds);
            }

            part.setDurationSeconds(durationSeconds);
            audioPartRepository.save(part);

            job.setStatus(JobStatus.COMPLETED);
            job.setResultPayload("{\"durationSeconds\":" + durationSeconds + "}");
            log.info("AUDIO_DURATION job {} completed: part {} = {}s", jobId, audioPartId, durationSeconds);
        } catch (Exception ex) {
            log.error("AUDIO_DURATION job {} failed: {}", jobId, ex.getMessage(), ex);
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(ex.getMessage());
        }

        job.setCompletedAt(Instant.now());
        jobRepository.save(job);
    }
}
```

- [ ] **Step 4: Run test**

```bash
./mvnw test -Dtest=AudioDurationJobRunnerTest
```

Expected: PASS

- [ ] **Step 5: Add @MockitoBean to AutomatationApplicationTests**

Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
AudioDurationJobRunner audioDurationJobRunner;
```

And the import:
```java
import kg.automation.rest.automatation.jobs.service.AudioDurationJobRunner;
```

Run:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/service/AudioDurationJobRunner.java \
        src/test/java/kg/automation/rest/automatation/jobs/service/AudioDurationJobRunnerTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: AudioDurationJobRunner — AUDIO_DURATION probes stage file and writes duration_seconds"
```

---

## Task 12: PipelineOrchestrator — audio phase transitions

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/pipeline/orchestrator/PipelineOrchestrator.java`
- Create: `src/test/java/kg/automation/rest/automatation/pipeline/orchestrator/PipelineOrchestratorAudioTest.java`

The orchestrator is the only place that decides "what job to create next." It reads DB state and emits `job_queue` rows. It does NOT call business services. It will be called by runners after success (via `advancePipeline(cycleId, pipelineRunId)`) and also by a scheduled poller.

- [ ] **Step 1: Write failing tests**

```java
// src/test/java/kg/automation/rest/automatation/pipeline/orchestrator/PipelineOrchestratorAudioTest.java
package kg.automation.rest.automatation.pipeline.orchestrator;

import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.PipelineRunEntity;
import kg.automation.rest.automatation.jobs.domain.PipelineRunStatus;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.PipelineRunRepository;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.domain.PartProcessingStatus;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class PipelineOrchestratorAudioTest {

    ProcessingJobRepository jobRepo = mock(ProcessingJobRepository.class);
    PipelineRunRepository runRepo = mock(PipelineRunRepository.class);
    AudioPartRepository partRepo = mock(AudioPartRepository.class);
    AudioStageRepository stageRepo = mock(AudioStageRepository.class);
    PipelineOrchestrator orchestrator;

    @BeforeEach
    void setUp() {
        orchestrator = new PipelineOrchestrator(jobRepo, runRepo, partRepo, stageRepo);
    }

    @Test
    void advanceAudio_allNoiseRemovedDone_createsTrimJobs() {
        long cycleId = 1L;
        UUID runId = UUID.randomUUID();

        // Two audio parts for this cycle
        AudioPartEntity part1 = buildPart(10L, cycleId, 1);
        AudioPartEntity part2 = buildPart(11L, cycleId, 2);

        // Both have NOISE_REMOVED stages
        AudioStageEntity noiseStage1 = buildStage(10L, AudioStageType.NOISE_REMOVED, "/tmp/p10_noise.mp3");
        AudioStageEntity noiseStage2 = buildStage(11L, AudioStageType.NOISE_REMOVED, "/tmp/p11_noise.mp3");

        // NOISE_REMOVE jobs both COMPLETED
        ProcessingJobEntity noiseJob1 = buildJob(JobType.AUDIO_NOISE_REMOVE, JobStatus.COMPLETED, cycleId, 10L, runId);
        ProcessingJobEntity noiseJob2 = buildJob(JobType.AUDIO_NOISE_REMOVE, JobStatus.COMPLETED, cycleId, 11L, runId);

        // No TRIM jobs yet
        PipelineRunEntity run = buildRun(runId, cycleId);

        when(runRepo.findById(runId)).thenReturn(Optional.of(run));
        when(runRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findByCycleId(cycleId)).thenReturn(List.of(part1, part2));
        when(stageRepo.findByAudioPartIdAndStage(10L, AudioStageType.NOISE_REMOVED)).thenReturn(Optional.of(noiseStage1));
        when(stageRepo.findByAudioPartIdAndStage(11L, AudioStageType.NOISE_REMOVED)).thenReturn(Optional.of(noiseStage2));
        when(jobRepo.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, JobType.AUDIO_NOISE_REMOVE, runId))
                .thenReturn(List.of(noiseJob1, noiseJob2));
        when(jobRepo.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, JobType.AUDIO_TRIM, runId))
                .thenReturn(List.of()); // not yet created
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));

        orchestrator.advanceAudio(cycleId, runId);

        // Verify two AUDIO_TRIM jobs were created
        verify(jobRepo, times(2)).save(argThat(j ->
                j instanceof ProcessingJobEntity &&
                ((ProcessingJobEntity) j).getJobType() == JobType.AUDIO_TRIM));
    }

    @Test
    void advanceAudio_allTrimmedDone_createsConcatAndDurationJobs() {
        long cycleId = 2L;
        UUID runId = UUID.randomUUID();

        AudioPartEntity part1 = buildPart(20L, cycleId, 1);

        AudioStageEntity trimStage = buildStage(20L, AudioStageType.TRIMMED, "/tmp/p20_trim.mp3");

        ProcessingJobEntity trimJob = buildJob(JobType.AUDIO_TRIM, JobStatus.COMPLETED, cycleId, 20L, runId);
        ProcessingJobEntity noiseJob = buildJob(JobType.AUDIO_NOISE_REMOVE, JobStatus.COMPLETED, cycleId, 20L, runId);

        PipelineRunEntity run = buildRun(runId, cycleId);

        when(runRepo.findById(runId)).thenReturn(Optional.of(run));
        when(runRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(partRepo.findByCycleId(cycleId)).thenReturn(List.of(part1));
        when(stageRepo.findByAudioPartIdAndStage(20L, AudioStageType.NOISE_REMOVED)).thenReturn(Optional.of(buildStage(20L, AudioStageType.NOISE_REMOVED, "/x")));
        when(stageRepo.findByAudioPartIdAndStage(20L, AudioStageType.TRIMMED)).thenReturn(Optional.of(trimStage));
        when(jobRepo.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, JobType.AUDIO_NOISE_REMOVE, runId))
                .thenReturn(List.of(noiseJob));
        when(jobRepo.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, JobType.AUDIO_TRIM, runId))
                .thenReturn(List.of(trimJob));
        when(jobRepo.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, JobType.AUDIO_CONCATENATE, runId))
                .thenReturn(List.of()); // not yet
        when(jobRepo.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, JobType.AUDIO_DURATION, runId))
                .thenReturn(List.of());
        when(jobRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));

        orchestrator.advanceAudio(cycleId, runId);

        // Verify AUDIO_CONCATENATE + AUDIO_DURATION jobs created
        verify(jobRepo, atLeastOnce()).save(argThat(j ->
                j instanceof ProcessingJobEntity &&
                (((ProcessingJobEntity) j).getJobType() == JobType.AUDIO_CONCATENATE
                        || ((ProcessingJobEntity) j).getJobType() == JobType.AUDIO_DURATION)));
    }

    // helpers
    private static AudioPartEntity buildPart(long id, long cycleId, int partNum) {
        AudioPartEntity p = new AudioPartEntity();
        p.setId(id);
        p.setCycleId(cycleId);
        p.setPartNumber(partNum);
        p.setProcessingStatus(PartProcessingStatus.IN_PROGRESS);
        p.setCreatedAt(Instant.now());
        p.setUpdatedAt(Instant.now());
        return p;
    }

    private static AudioStageEntity buildStage(long audioPartId, AudioStageType type, String path) {
        AudioStageEntity s = new AudioStageEntity();
        s.setAudioPartId(audioPartId);
        s.setStage(type);
        s.setFilePath(path);
        s.setCreatedAt(Instant.now());
        return s;
    }

    private static ProcessingJobEntity buildJob(JobType type, JobStatus status, long cycleId, long audioPartId, UUID runId) {
        ProcessingJobEntity j = new ProcessingJobEntity();
        j.setId(UUID.randomUUID());
        j.setJobType(type);
        j.setStatus(status);
        j.setCycleId(cycleId);
        j.setAudioPartId(audioPartId);
        j.setPipelineRunId(runId);
        j.setCreatedAt(Instant.now());
        j.setUpdatedAt(Instant.now());
        j.setRetryCount(0);
        return j;
    }

    private static PipelineRunEntity buildRun(UUID id, long cycleId) {
        PipelineRunEntity r = new PipelineRunEntity();
        r.setId(id);
        r.setCycleId(cycleId);
        r.setStatus(PipelineRunStatus.RUNNING);
        r.setCurrentPhase("AUDIO_NOISE_REMOVE");
        r.setCreatedAt(Instant.now());
        r.setUpdatedAt(Instant.now());
        return r;
    }
}
```

- [ ] **Step 2: Add query methods to ProcessingJobRepository**

In `ProcessingJobRepository.java`:

```java
List<ProcessingJobEntity> findByCycleIdAndJobTypeAndPipelineRunId(
        Long cycleId, JobType jobType, UUID pipelineRunId);

List<ProcessingJobEntity> findByCycleIdAndJobType(Long cycleId, JobType jobType);
```

Also add to `AudioPartRepository.java`:

```java
List<AudioPartEntity> findByCycleId(Long cycleId);
```

- [ ] **Step 3: Run to verify failure**

```bash
./mvnw test -Dtest=PipelineOrchestratorAudioTest
```

Expected: FAIL — `PipelineOrchestrator not found`

- [ ] **Step 4: Create PipelineOrchestrator**

```java
// src/main/java/kg/automation/rest/automatation/pipeline/orchestrator/PipelineOrchestrator.java
package kg.automation.rest.automatation.pipeline.orchestrator;

import kg.automation.rest.automatation.jobs.domain.JobStatus;
import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.domain.PipelineRunEntity;
import kg.automation.rest.automatation.jobs.domain.PipelineRunStatus;
import kg.automation.rest.automatation.jobs.domain.ProcessingJobEntity;
import kg.automation.rest.automatation.jobs.repository.PipelineRunRepository;
import kg.automation.rest.automatation.jobs.repository.ProcessingJobRepository;
import kg.automation.rest.automatation.pipeline.domain.AudioPartEntity;
import kg.automation.rest.automatation.pipeline.domain.AudioStageType;
import kg.automation.rest.automatation.pipeline.repository.AudioPartRepository;
import kg.automation.rest.automatation.pipeline.repository.AudioStageRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

/**
 * The single place that decides "what job to create next" for a pipeline run.
 * Does NOT call business services — only reads and writes to job_queue and pipeline_run.
 *
 * Phase transitions (audio pipeline only in Plan A):
 *   Phase 1: All AUDIO_NOISE_REMOVE completed + all parts have NOISE_REMOVED stage
 *            → create AUDIO_TRIM per part
 *   Phase 2: All AUDIO_TRIM completed + all parts have TRIMMED stage
 *            → create AUDIO_CONCATENATE (one job for cycle) + AUDIO_DURATION per part
 *   Phase 3: AUDIO_CONCATENATE completed
 *            → create AUDIO_COMPRESS for the concatenated part
 *   Phase 4: AUDIO_COMPRESS completed
 *            → update pipeline_run.current_phase = "AUDIO_COMPLETE" (Plan B picks up from here)
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class PipelineOrchestrator {

    private final ProcessingJobRepository jobRepository;
    private final PipelineRunRepository pipelineRunRepository;
    private final AudioPartRepository audioPartRepository;
    private final AudioStageRepository audioStageRepository;

    @Transactional
    public void advanceAudio(long cycleId, UUID pipelineRunId) {
        PipelineRunEntity run = pipelineRunRepository.findById(pipelineRunId)
                .orElseThrow(() -> new IllegalStateException("pipeline_run not found: " + pipelineRunId));

        if (run.getStatus() == PipelineRunStatus.COMPLETED
                || run.getStatus() == PipelineRunStatus.FAILED
                || run.getStatus() == PipelineRunStatus.CANCELLED) {
            log.debug("pipeline_run {} is terminal ({}), nothing to advance", pipelineRunId, run.getStatus());
            return;
        }

        List<AudioPartEntity> parts = audioPartRepository.findByCycleId(cycleId);
        if (parts.isEmpty()) {
            log.warn("No audio_part rows for cycle {}, cannot advance audio pipeline", cycleId);
            return;
        }

        // Phase 1 → Phase 2: all NOISE_REMOVE done, no TRIM yet
        if (allJobsCompleted(cycleId, JobType.AUDIO_NOISE_REMOVE, pipelineRunId)
                && allPartsHaveStage(parts, AudioStageType.NOISE_REMOVED)
                && noJobsExist(cycleId, JobType.AUDIO_TRIM, pipelineRunId)) {

            log.info("Cycle {} run {}: NOISE_REMOVE phase complete → creating AUDIO_TRIM jobs", cycleId, pipelineRunId);
            for (AudioPartEntity part : parts) {
                createJob(JobType.AUDIO_TRIM, cycleId, part.getId(), null, null, pipelineRunId);
            }
            run.setCurrentPhase("AUDIO_TRIM");
            pipelineRunRepository.save(run);
            return;
        }

        // Phase 2 → Phase 3: all TRIM done, no CONCATENATE yet
        if (allJobsCompleted(cycleId, JobType.AUDIO_TRIM, pipelineRunId)
                && allPartsHaveStage(parts, AudioStageType.TRIMMED)
                && noJobsExist(cycleId, JobType.AUDIO_CONCATENATE, pipelineRunId)) {

            log.info("Cycle {} run {}: TRIM phase complete → creating AUDIO_CONCATENATE + AUDIO_DURATION", cycleId, pipelineRunId);
            // AUDIO_CONCATENATE: uses audio_part_id of the first part (produces full-book concat)
            long firstPartId = parts.stream().mapToLong(AudioPartEntity::getId).min().orElse(parts.get(0).getId());
            createJob(JobType.AUDIO_CONCATENATE, cycleId, firstPartId, null, null, pipelineRunId);
            // AUDIO_DURATION per part (to record per-chapter duration)
            for (AudioPartEntity part : parts) {
                createJob(JobType.AUDIO_DURATION, cycleId, part.getId(), null, null, pipelineRunId);
            }
            run.setCurrentPhase("AUDIO_CONCATENATE");
            pipelineRunRepository.save(run);
            return;
        }

        // Phase 3 → Phase 4: CONCATENATE done, no COMPRESS yet
        if (allJobsCompleted(cycleId, JobType.AUDIO_CONCATENATE, pipelineRunId)
                && noJobsExist(cycleId, JobType.AUDIO_COMPRESS, pipelineRunId)) {

            log.info("Cycle {} run {}: CONCATENATE complete → creating AUDIO_COMPRESS", cycleId, pipelineRunId);
            List<ProcessingJobEntity> concatJobs = jobRepository
                    .findByCycleIdAndJobTypeAndPipelineRunId(cycleId, JobType.AUDIO_CONCATENATE, pipelineRunId);
            long concatPartId = concatJobs.get(0).getAudioPartId();
            createJob(JobType.AUDIO_COMPRESS, cycleId, concatPartId, null, null, pipelineRunId);
            run.setCurrentPhase("AUDIO_COMPRESS");
            pipelineRunRepository.save(run);
            return;
        }

        // Phase 4: COMPRESS done → audio complete
        if (allJobsCompleted(cycleId, JobType.AUDIO_COMPRESS, pipelineRunId)
                && !"AUDIO_COMPLETE".equals(run.getCurrentPhase())) {

            log.info("Cycle {} run {}: COMPRESS complete → audio pipeline done", cycleId, pipelineRunId);
            run.setCurrentPhase("AUDIO_COMPLETE");
            pipelineRunRepository.save(run);
            // Plan B picks up from AUDIO_COMPLETE to schedule IMAGE_GENERATE + VIDEO_RENDER
        }
    }

    private boolean allJobsCompleted(long cycleId, JobType type, UUID runId) {
        List<ProcessingJobEntity> jobs = jobRepository.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, type, runId);
        return !jobs.isEmpty() && jobs.stream().allMatch(j -> j.getStatus() == JobStatus.COMPLETED);
    }

    private boolean noJobsExist(long cycleId, JobType type, UUID runId) {
        return jobRepository.findByCycleIdAndJobTypeAndPipelineRunId(cycleId, type, runId).isEmpty();
    }

    private boolean allPartsHaveStage(List<AudioPartEntity> parts, AudioStageType stage) {
        return parts.stream().allMatch(p ->
                audioStageRepository.findByAudioPartIdAndStage(p.getId(), stage).isPresent());
    }

    private ProcessingJobEntity createJob(
            JobType type, long cycleId, Long audioPartId,
            Long videoPartId, Long imageAssetId, UUID pipelineRunId) {
        ProcessingJobEntity job = new ProcessingJobEntity();
        job.setJobType(type);
        job.setStatus(JobStatus.PENDING);
        job.setCycleId(cycleId);
        job.setAudioPartId(audioPartId);
        job.setVideoPartId(videoPartId);
        job.setImageAssetId(imageAssetId);
        job.setPipelineRunId(pipelineRunId);
        job.setRetryCount(0);
        job.setCreatedAt(Instant.now());
        job.setUpdatedAt(Instant.now());
        return jobRepository.save(job);
    }
}
```

- [ ] **Step 5: Run test**

```bash
./mvnw test -Dtest=PipelineOrchestratorAudioTest
```

Expected: PASS

- [ ] **Step 6: Add @MockitoBean to AutomatationApplicationTests**

Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
PipelineOrchestrator pipelineOrchestrator;
```

And the import:
```java
import kg.automation.rest.automatation.pipeline.orchestrator.PipelineOrchestrator;
```

Run:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/pipeline/orchestrator/PipelineOrchestrator.java \
        src/main/java/kg/automation/rest/automatation/jobs/repository/ProcessingJobRepository.java \
        src/main/java/kg/automation/rest/automatation/pipeline/repository/AudioPartRepository.java \
        src/test/java/kg/automation/rest/automatation/pipeline/orchestrator/PipelineOrchestratorAudioTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: PipelineOrchestrator — DB-driven audio phase transitions (noise-remove→trim→concat→compress)"
```

---

## Task 13: Extend PipelineJobsRabbitListener for new job types

**Files:**
- Modify: `src/main/java/kg/automation/rest/automatation/infra/rabbit/PipelineJobsRabbitListener.java`
- Modify: `src/main/java/kg/automation/rest/automatation/jobs/runtime/AudioJobExecutor.java`

- [ ] **Step 1: Add generic dispatch to AudioJobExecutor**

Replace `enqueueConcatenate` with a generic `enqueue(JobType, UUID)` method. Keep `enqueueConcatenate` as a delegate to avoid breaking existing callers.

```java
package kg.automation.rest.automatation.jobs.runtime;

import kg.automation.rest.automatation.jobs.domain.JobType;
import kg.automation.rest.automatation.jobs.service.AudioCompressJobRunner;
import kg.automation.rest.automatation.jobs.service.AudioDurationJobRunner;
import kg.automation.rest.automatation.jobs.service.AudioNoiseRemoveJobRunner;
import kg.automation.rest.automatation.jobs.service.AudioTrimJobRunner;
import kg.automation.rest.automatation.jobs.service.ConcatenateProcessingJobRunner;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.core.task.TaskExecutor;
import org.springframework.stereotype.Component;

import java.util.UUID;

@Component
public class AudioJobExecutor {

    private final TaskExecutor audioProcessingExecutor;
    private final ConcatenateProcessingJobRunner concatenateJobRunner;
    private final AudioNoiseRemoveJobRunner noiseRemoveJobRunner;
    private final AudioTrimJobRunner trimJobRunner;
    private final AudioCompressJobRunner compressJobRunner;
    private final AudioDurationJobRunner durationJobRunner;

    public AudioJobExecutor(
            @Qualifier("audioProcessingExecutor") TaskExecutor audioProcessingExecutor,
            ConcatenateProcessingJobRunner concatenateJobRunner,
            AudioNoiseRemoveJobRunner noiseRemoveJobRunner,
            AudioTrimJobRunner trimJobRunner,
            AudioCompressJobRunner compressJobRunner,
            AudioDurationJobRunner durationJobRunner) {
        this.audioProcessingExecutor = audioProcessingExecutor;
        this.concatenateJobRunner = concatenateJobRunner;
        this.noiseRemoveJobRunner = noiseRemoveJobRunner;
        this.trimJobRunner = trimJobRunner;
        this.compressJobRunner = compressJobRunner;
        this.durationJobRunner = durationJobRunner;
    }

    /** Submits {@code jobId} to the local thread-pool executor for the given job type. */
    public void enqueue(JobType jobType, UUID jobId) {
        switch (jobType) {
            case AUDIO_CONCATENATE -> audioProcessingExecutor.execute(() -> concatenateJobRunner.run(jobId));
            case AUDIO_NOISE_REMOVE -> audioProcessingExecutor.execute(() -> noiseRemoveJobRunner.run(jobId));
            case AUDIO_TRIM -> audioProcessingExecutor.execute(() -> trimJobRunner.run(jobId));
            case AUDIO_COMPRESS -> audioProcessingExecutor.execute(() -> compressJobRunner.run(jobId));
            case AUDIO_DURATION -> audioProcessingExecutor.execute(() -> durationJobRunner.run(jobId));
            default -> throw new IllegalArgumentException("Unsupported job type for local executor: " + jobType);
        }
    }

    /** Convenience method retained for existing callers. */
    public void enqueueConcatenate(UUID jobId) {
        enqueue(JobType.AUDIO_CONCATENATE, jobId);
    }
}
```

- [ ] **Step 2: Update PipelineJobsRabbitListener — split into two @RabbitListener methods**

The `audio.noise-remove` queue must use `noiseRemoveListenerContainerFactory` (prefetchCount=1) for GPU
protection. It cannot be in the same `@RabbitListener` as the priority queues, which use the default
factory. Split into two methods in `PipelineJobsRabbitListener.java`:

```java
/** Handles all non-GPU audio jobs from the three priority queues. */
@RabbitListener(
        queues = {
            InfraRabbitConfiguration.QUEUE_PROCESSING_HIGH,
            InfraRabbitConfiguration.QUEUE_PROCESSING_NORMAL,
            InfraRabbitConfiguration.QUEUE_PROCESSING_LOW
        },
        containerFactory = "rabbitListenerContainerFactory")
public void onMessage(PipelineJobMessage message) {
    if (message == null || message.getProcessingJobId() == null) {
        log.warn("Ignoring invalid pipeline message: {}", message);
        return;
    }
    JobType jobType = message.getJobType();
    if (jobType == null) {
        log.warn("Null jobType in pipeline message id={}", message.getMessageId());
        return;
    }
    boolean isAudioJob = switch (jobType) {
        case AUDIO_CONCATENATE, AUDIO_TRIM, AUDIO_COMPRESS, AUDIO_DURATION -> true;
        default -> false;
    };
    if (!isAudioJob) {
        log.warn("Unsupported job type on pipeline queues, id={}: {}", message.getProcessingJobId(), jobType);
        return;
    }
    dispatch(message, jobType);
}

/**
 * Handles AUDIO_NOISE_REMOVE jobs from the dedicated GPU queue.
 * Uses noiseRemoveListenerContainerFactory (prefetchCount=1) so only one Demucs call
 * runs at a time, protecting GPU memory.
 */
@RabbitListener(
        queues = InfraRabbitConfiguration.QUEUE_AUDIO_NOISE_REMOVE,
        containerFactory = "noiseRemoveListenerContainerFactory")
public void onNoiseRemoveMessage(PipelineJobMessage message) {
    if (message == null || message.getProcessingJobId() == null) {
        log.warn("Ignoring invalid noise-remove message: {}", message);
        return;
    }
    dispatch(message, JobType.AUDIO_NOISE_REMOVE);
}

private void dispatch(PipelineJobMessage message, JobType jobType) {
    pipelineMessageAuditBridge.recordConsumeStart(message, CONSUMER_ID);
    log.debug("Running {} job {} (correlationId={}, messageId={})",
            jobType, message.getProcessingJobId(), message.getCorrelationId(), message.getMessageId());

    audioJobExecutor.enqueue(jobType, message.getProcessingJobId());

    processingJobRepository.findById(message.getProcessingJobId()).ifPresentOrElse(
            job -> {
                if (job.getStatus() == JobStatus.FAILED) {
                    pipelineMessageAuditBridge.recordExecuteOutcome(
                            message.getMessageId(), "FAILED", job.getErrorMessage());
                } else {
                    pipelineMessageAuditBridge.recordExecuteOutcome(
                            message.getMessageId(), "COMPLETED", null);
                }
            },
            () -> pipelineMessageAuditBridge.recordExecuteOutcome(
                    message.getMessageId(), "FAILED", "processing job row missing"));
}
```

Also update the constructor injection to include `AudioJobExecutor` instead of `ConcatenateProcessingJobRunner` directly:

```java
private final AudioJobExecutor audioJobExecutor;
private final ProcessingJobRepository processingJobRepository;
private final PipelineMessageAuditBridge pipelineMessageAuditBridge;
```

- [ ] **Step 3: Run all tests**

```bash
./mvnw test
```

Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 4: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/runtime/AudioJobExecutor.java \
        src/main/java/kg/automation/rest/automatation/infra/rabbit/PipelineJobsRabbitListener.java
git commit -m "feat: AudioJobExecutor + listener dispatch all audio job types (NOISE_REMOVE, TRIM, COMPRESS, DURATION)"
```

---

## Task 14: Final verification

All `@MockitoBean` entries were added incrementally in Tasks 5–12. This task is a final sanity check.

- [ ] **Step 1: Verify AutomatationApplicationTests has all required mocks**

Confirm `AutomatationApplicationTests.java` contains `@MockitoBean` for:
- `SilenceServiceClient` (added in Task 5)
- `WhisperServiceClient` (added in Task 6)
- `AudioNoiseRemoveJobRunner` (added in Task 8)
- `AudioTrimJobRunner` (added in Task 9)
- `AudioCompressJobRunner` (added in Task 10)
- `AudioDurationJobRunner` (added in Task 11)
- `PipelineOrchestrator` (added in Task 12)

If any are missing, add them now following the same pattern as the others.

- [ ] **Step 2: Run the full test suite**

```bash
./mvnw test
```

Expected: BUILD SUCCESS — zero failures

- [ ] **Step 3: Confirm definition of done**

All items in the "Definition of done" section below must be ✅ before marking Plan A complete.

---

## Definition of done for Plan A

- [ ] `./mvnw test` passes with zero failures
- [ ] `V202605080001` migration applies cleanly in `FlywayCycleSchemaIT`
- [ ] `pipeline_run` table and `job_queue.pipeline_run_id` exist in DB
- [ ] `JobType` has all 9 values
- [ ] `SilenceServiceClient` and `WhisperServiceClient` have unit tests with `MockRestServiceServer`
- [ ] All four audio runners (`AudioNoiseRemoveJobRunner`, `AudioTrimJobRunner`, `AudioCompressJobRunner`, `AudioDurationJobRunner`) have unit tests verifying COMPLETED path and FAILED path
- [ ] `PipelineOrchestrator.advanceAudio()` has unit tests for Phase 1→2 and Phase 2→3 transitions
- [ ] `PipelineJobsRabbitListener` dispatches all audio job types without falling through to the "unsupported" warning
- [ ] `AutomatationApplicationTests.contextLoads()` passes

**Continue with Plan B** (`docs/superpowers/plans/2026-05-08-pipeline-plan-b-image-video-distribution.md`) for image generation, video rendering, distribution runners, and the full orchestrator phase transitions (including image/video/distribution phases).
