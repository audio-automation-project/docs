# Pipeline Orchestrator — Plan B: Image, Video & Distribution

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the pipeline from Plan A's `AUDIO_COMPLETE` phase through image generation (AI-driven), video rendering, and multi-platform distribution. Introduce pluggable AI provider interfaces for both text (prompt generation) and image (cover generation), config-driven provider selection, particle overlay management, and a per-platform distribution strategy with partial-failure handling.

**Architecture:** After `AUDIO_COMPLETE`, the orchestrator creates one `IMAGE_GENERATE` job per cycle. That runner calls `AiTextProvider` (prompt from book description) → `AiImageProvider` (base cover) → `BookCoverService` (per-part covers). Then `VIDEO_RENDER` jobs render each part. Finally one `DISTRIBUTE` job fans out to all target platforms via `PlatformUploader` strategies; per-platform failures are isolated in `distribution_result` and individually retryable.

**Tech Stack:** Same as Plan A — Spring Boot 3.4, Java 17, Spring AMQP, Flyway, JPA. New: pluggable AI gateways via `RestTemplate`.

**Prerequisite:** Plan A Tasks 1–13 committed (branch `pipeline-plan-a`).

---

## File map

**New files:**
- `src/main/resources/db/migration/V202605080002__overlay_asset.sql` — `overlay_asset` table
- `src/main/java/.../ai/provider/AiTextProvider.java` — text AI strategy interface
- `src/main/java/.../ai/provider/AiImageProvider.java` — image AI strategy interface
- `src/main/java/.../ai/provider/AiProviderConfiguration.java` — config-driven bean registration
- `src/main/java/.../ai/provider/OllamaTextProvider.java` — Ollama implementation of `AiTextProvider`
- `src/main/java/.../ai/provider/StubImageProvider.java` — stub `AiImageProvider` (placeholder)
- `src/main/java/.../pipeline/domain/OverlayAssetEntity.java` — JPA entity for particle overlays
- `src/main/java/.../pipeline/repository/OverlayAssetRepository.java`
- `src/main/java/.../pipeline/service/OverlayStorageService.java` — file save + DB persistence
- `src/main/java/.../pipeline/web/OverlayController.java` — REST upload endpoint
- `src/main/java/.../jobs/service/ImageGenerateJobRunner.java`
- `src/main/java/.../jobs/service/VideoRenderJobRunner.java`
- `src/main/java/.../jobs/service/DistributeJobRunner.java`
- `src/main/java/.../poster/PlatformUploader.java` — strategy interface
- `src/main/java/.../poster/PlatformUploadRequest.java` — input DTO
- `src/main/java/.../poster/PlatformUploadResult.java` — output DTO
- `src/main/java/.../poster/BoostyPlatformUploader.java` — wraps existing `Poster`
- `src/main/java/.../poster/YouTubePlatformUploader.java` — wraps existing `YouTubeUploader` (stub)
- `src/test/java/.../ai/provider/OllamaTextProviderTest.java`
- `src/test/java/.../jobs/service/ImageGenerateJobRunnerTest.java`
- `src/test/java/.../jobs/service/VideoRenderJobRunnerTest.java`
- `src/test/java/.../jobs/service/DistributeJobRunnerTest.java`
- `src/test/java/.../pipeline/orchestrator/PipelineOrchestratorPostAudioTest.java`

**Modified files:**
- `src/main/java/.../pipeline/orchestrator/PipelineOrchestrator.java` — add `advanceImageVideo()`, `advanceDistribution()`, top-level `advance()`
- `src/main/java/.../jobs/runtime/AudioJobExecutor.java` — rename concept to `PipelineJobExecutor`, add IMAGE_GENERATE, VIDEO_RENDER, DISTRIBUTE
- `src/main/java/.../infra/rabbit/PipelineJobsRabbitListener.java` — accept new job types in `onMessage`
- `src/main/java/.../pipeline/repository/ImageAssetRepository.java` — add `findByCycleIdAndAssetType`
- `src/main/java/.../pipeline/repository/VideoPartRepository.java` — add `findByCycleId`
- `src/test/java/.../AutomatationApplicationTests.java` — add mocks for new beans

---

## Recommended task order

```
1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12 → 13
```

Rationale:
- **Tasks 1–3** build the AI provider abstraction (interfaces → config → Ollama implementation) — needed before `ImageGenerateJobRunner`.
- **Task 4** builds overlay storage — needed before `VideoRenderJobRunner`.
- **Task 5** is the stub image provider — needed so `ImageGenerateJobRunner` compiles.
- **Tasks 6–8** are the three runners (image → video → distribute), each building on the prior.
- **Tasks 9–10** are the platform uploader strategies — needed by `DistributeJobRunner`.
- **Task 11** extends the orchestrator with post-audio phase transitions.
- **Task 12** wires the executor and listener for new job types.
- **Task 13** is final verification.

---

## Task 1: AiTextProvider and AiImageProvider interfaces

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/ai/provider/AiTextProvider.java`
- Create: `src/main/java/kg/automation/rest/automatation/ai/provider/AiImageProvider.java`

- [ ] **Step 1: Create AiTextProvider interface**

```java
package kg.automation.rest.automatation.ai.provider;

/**
 * Strategy interface for text generation AI.
 * Implementations: Ollama, OpenAI, Gemini, local LLM.
 * Selected via {@code ai.text.provider} config property.
 */
public interface AiTextProvider {

    /**
     * Generates a creative image-generation prompt from book metadata.
     *
     * @param bookTitle       the book title
     * @param bookDescription the book description / synopsis
     * @return a detailed prompt suitable for an image-generation AI
     */
    String generateImagePrompt(String bookTitle, String bookDescription);
}
```

- [ ] **Step 2: Create AiImageProvider interface**

```java
package kg.automation.rest.automatation.ai.provider;

/**
 * Strategy interface for image generation AI.
 * Implementations: Gemini, OpenAI, Midjourney, etc.
 * Selected via {@code ai.image.provider} config property.
 */
public interface AiImageProvider {

    /**
     * Generates an image from the given prompt.
     *
     * @param prompt the image-generation prompt (output of {@link AiTextProvider#generateImagePrompt})
     * @return raw image bytes (PNG or JPEG)
     */
    byte[] generateImage(String prompt);

    /**
     * @return the MIME type of the generated image (e.g. "image/png")
     */
    default String mimeType() {
        return "image/png";
    }
}
```

- [ ] **Step 3: Verify compilation**

```bash
./mvnw compile -q
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/ai/provider/AiTextProvider.java \
        src/main/java/kg/automation/rest/automatation/ai/provider/AiImageProvider.java
git commit -m "feat: AiTextProvider + AiImageProvider strategy interfaces for pluggable AI"
```

---

## Task 2: AiProviderConfiguration — config-driven bean registration

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/ai/provider/AiProviderConfiguration.java`

- [ ] **Step 1: Create configuration class**

```java
package kg.automation.rest.automatation.ai.provider;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

/**
 * Registers the active {@link AiTextProvider} and {@link AiImageProvider} beans
 * based on {@code ai.text.provider} and {@code ai.image.provider} config properties.
 *
 * Add new {@code @Bean @ConditionalOnProperty} methods as new providers are implemented.
 */
@Configuration
public class AiProviderConfiguration {

    @Bean
    @ConditionalOnProperty(name = "ai.text.provider", havingValue = "ollama", matchIfMissing = true)
    public AiTextProvider ollamaTextProvider(RestTemplate restTemplate,
            @org.springframework.beans.factory.annotation.Value("${ai.text.ollama.url:http://localhost:8080/api/chat/completions}") String ollamaUrl,
            @org.springframework.beans.factory.annotation.Value("${ai.text.ollama.model:gemma:latest}") String model) {
        return new OllamaTextProvider(restTemplate, ollamaUrl, model);
    }

    @Bean
    @ConditionalOnProperty(name = "ai.image.provider", havingValue = "stub", matchIfMissing = true)
    public AiImageProvider stubImageProvider() {
        return new StubImageProvider();
    }
}
```

- [ ] **Step 2: Verify compilation**

```bash
./mvnw compile -q
```

Expected: FAIL — `OllamaTextProvider` and `StubImageProvider` not yet created (expected, will be created in Tasks 3 and 5)

- [ ] **Step 3: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/ai/provider/AiProviderConfiguration.java
git commit -m "feat: AiProviderConfiguration — config-driven AI provider bean registration"
```

---

## Task 3: OllamaTextProvider implementation

**Files:**
- Create: `src/main/java/kg/automation/rest/automatation/ai/provider/OllamaTextProvider.java`
- Create: `src/test/java/kg/automation/rest/automatation/ai/provider/OllamaTextProviderTest.java`

- [ ] **Step 1: Write failing test**

```java
package kg.automation.rest.automatation.ai.provider;

import org.junit.jupiter.api.Test;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.test.web.client.MockRestServiceServer;
import org.springframework.web.client.RestTemplate;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.method;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo;
import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;

class OllamaTextProviderTest {

    private final RestTemplate restTemplate = new RestTemplate();
    private final MockRestServiceServer mockServer = MockRestServiceServer.createServer(restTemplate);
    private final OllamaTextProvider provider = new OllamaTextProvider(
            restTemplate, "http://localhost:8080/api/chat/completions", "gemma:latest");

    @Test
    void generateImagePrompt_returnsPromptText() {
        String responseBody = """
                {"id":"1","model":"gemma:latest","choices":{"index":0,"logprobs":false,"finish_reason":"stop","message":{"role":"assistant","content":"A mystical forest cover with golden light"}}}
                """;
        mockServer.expect(requestTo("http://localhost:8080/api/chat/completions"))
                  .andExpect(method(HttpMethod.POST))
                  .andRespond(withSuccess(responseBody, MediaType.APPLICATION_JSON));

        String result = provider.generateImagePrompt("Dark Forest", "A hero enters a mystical forest");
        assertThat(result).contains("mystical forest");
        mockServer.verify();
    }

    @Test
    void generateImagePrompt_nullTitle_throwsIllegalArgument() {
        assertThatThrownBy(() -> provider.generateImagePrompt(null, "desc"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Create OllamaTextProvider**

```java
package kg.automation.rest.automatation.ai.provider;

import kg.automation.rest.automatation.ai.Message;
import kg.automation.rest.automatation.ai.ollama.LlamaMessage;
import kg.automation.rest.automatation.ai.ollama.LlamaRespond;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.web.client.RestTemplate;

/**
 * {@link AiTextProvider} backed by a local Ollama instance.
 * Calls the chat-completions endpoint and extracts the assistant message content.
 */
@Slf4j
public class OllamaTextProvider implements AiTextProvider {

    private static final String SYSTEM_PROMPT = """
            You are an expert artist and AI image-prompt writer.
            Given a book title and description, generate a vivid 200-word image prompt
            that describes the ideal audiobook cover art.
            Include style references (e.g. "like a sci-fi novel cover from the 1980s").
            Output ONLY the image prompt, no explanations.
            """;

    private final RestTemplate restTemplate;
    private final String ollamaUrl;
    private final String model;

    public OllamaTextProvider(RestTemplate restTemplate, String ollamaUrl, String model) {
        this.restTemplate = restTemplate;
        this.ollamaUrl = ollamaUrl;
        this.model = model;
    }

    @Override
    public String generateImagePrompt(String bookTitle, String bookDescription) {
        if (bookTitle == null || bookTitle.isBlank()) {
            throw new IllegalArgumentException("bookTitle is required");
        }
        String userContent = "Title: " + bookTitle + "\nDescription: "
                + (bookDescription != null ? bookDescription : "No description available");

        LlamaMessage request = LlamaMessage.builder()
                .model(model)
                .message(new Message("user", SYSTEM_PROMPT + "\n\n" + userContent))
                .build();

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<LlamaMessage> entity = new HttpEntity<>(request, headers);

        log.info("Calling Ollama at {} for image prompt (model={})", ollamaUrl, model);
        LlamaRespond response = restTemplate.postForObject(ollamaUrl, entity, LlamaRespond.class);

        if (response == null || response.getChoices() == null
                || response.getChoices().getMessage() == null
                || response.getChoices().getMessage().getContent() == null) {
            throw new IllegalStateException("Ollama returned empty or malformed response");
        }

        String prompt = response.getChoices().getMessage().getContent().trim();
        log.info("Ollama generated image prompt ({} chars)", prompt.length());
        return prompt;
    }
}
```

- [ ] **Step 3: Run tests**

```bash
./mvnw test -Dtest=OllamaTextProviderTest
```

Expected: PASS

- [ ] **Step 4: Add @MockitoBean to AutomatationApplicationTests**

Add to `AutomatationApplicationTests.java`:

```java
@MockitoBean
AiTextProvider aiTextProvider;
@MockitoBean
AiImageProvider aiImageProvider;
```

And imports:
```java
import kg.automation.rest.automatation.ai.provider.AiTextProvider;
import kg.automation.rest.automatation.ai.provider.AiImageProvider;
```

Run:
```bash
./mvnw test -Dtest=AutomatationApplicationTests
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/ai/provider/OllamaTextProvider.java \
        src/test/java/kg/automation/rest/automatation/ai/provider/OllamaTextProviderTest.java \
        src/test/java/kg/automation/rest/automatation/AutomatationApplicationTests.java
git commit -m "feat: OllamaTextProvider — Ollama-backed AiTextProvider for image prompt generation"
```

---

## Task 4: Flyway — overlay_asset table

**Files:**
- Create: `src/main/resources/db/migration/V202605080002__overlay_asset.sql`

- [ ] **Step 1: Write the migration**

```sql
-- Particle overlay assets for video rendering.
-- Uploaded via REST, referenced by name from video render jobs.

CREATE TABLE overlay_asset (
    id          BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    file_path   TEXT NOT NULL,
    mime_type   VARCHAR(128) NOT NULL DEFAULT 'video/quicktime',
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TRIGGER trg_overlay_asset_updated_at
    BEFORE UPDATE ON overlay_asset
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Default overlay name on cycle; VideoRenderJobRunner reads this instead of inputPayload.
ALTER TABLE cycle ADD COLUMN overlay_name TEXT NOT NULL DEFAULT 'default';
```

> **Entity update:** Add `@Column(name = "overlay_name") private String overlayName = "default";` to `CycleEntity.java`.

- [ ] **Step 2: Run Flyway test**

```bash
./mvnw test -Dtest=FlywayCycleSchemaIT -pl .
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add src/main/resources/db/migration/V202605080002__overlay_asset.sql
git commit -m "feat: overlay_asset table for particle overlay management"
```

---

## Task 5: StubImageProvider + OverlayAssetEntity + OverlayStorageService + OverlayController

**New files:**
- `src/main/java/.../ai/provider/StubImageProvider.java`
- `src/main/java/.../pipeline/domain/OverlayAssetEntity.java`
- `src/main/java/.../pipeline/repository/OverlayAssetRepository.java`
- `src/main/java/.../pipeline/service/OverlayStorageService.java`
- `src/main/java/.../pipeline/web/OverlayController.java`

- [ ] **Step 1: StubImageProvider** — throws `UnsupportedOperationException("No image provider configured. Set ai.image.provider to a real provider.")`. Implements `AiImageProvider`.

- [ ] **Step 2: OverlayAssetEntity** — `@Entity @Table(name="overlay_asset")` with fields: `Long id` (IDENTITY), `String name` (unique), `String filePath`, `String mimeType`, `String description`, `Instant createdAt`, `Instant updatedAt`. `@PrePersist`/`@PreUpdate` for timestamps.

- [ ] **Step 3: OverlayAssetRepository** — `JpaRepository<OverlayAssetEntity, Long>` with `Optional<OverlayAssetEntity> findByName(String name)`.

- [ ] **Step 4: OverlayStorageService** — `@Service @RequiredArgsConstructor`:
  - Inject `OverlayAssetRepository` + `@Value("${processing.temp.dir}")` base path
  - `public OverlayAssetEntity store(String name, byte[] data, String mimeType, String description)`:
    1. Save file to `{base}/overlays/{name}.mov` (create dirs)
    2. Upsert `OverlayAssetEntity` row
    3. Return entity
  - `public Path resolve(String name)` — lookup by name, return `Path.of(entity.getFilePath())`, throw if not found

- [ ] **Step 5: OverlayController** — `@RestController @RequestMapping("/api/v1/overlays")`:
  - `@PostMapping(consumes = MULTIPART_FORM_DATA_VALUE)` accepting `@RequestParam("file") MultipartFile`, `@RequestParam("name") String`, optional `@RequestParam("description")`. Calls `overlayStorageService.store(...)`. Returns `ResponseEntity.ok(entity)`.

- [ ] **Step 6: Verify + Commit**

```bash
./mvnw compile -q
git add -A && git commit -m "feat: StubImageProvider + overlay_asset entity, storage, REST upload"
```

---

## Task 6: Repository additions + ImageGenerateJobRunner

**Modified:** `ImageAssetRepository` — add `findByCycleIdAndAssetType(Long cycleId, ImageAssetType type)`
**New:** `src/main/java/.../jobs/service/ImageGenerateJobRunner.java`
**New:** `src/test/java/.../jobs/service/ImageGenerateJobRunnerTest.java`

- [ ] **Step 1: Add query to ImageAssetRepository**

```java
List<ImageAssetEntity> findByCycleIdAndAssetType(Long cycleId, ImageAssetType assetType);
```

- [ ] **Step 2: ImageGenerateJobRunner** — `@Component @RequiredArgsConstructor`:

> **Ownership:** IMAGE_GENERATE is a **cycle-owned** job. The `job_queue` row has `cycle_id` set and `image_asset_id = NULL`. The runner creates the `image_asset` rows itself — no pre-creation needed because the base image doesn't exist before the AI call.

Constructor-injected deps: `ProcessingJobRepository`, `CycleRepository`, `BookRepository`, `AudioPartRepository`, `ImageAssetRepository`, `AiTextProvider`, `AiImageProvider`, `ImagePayloadService`, `BookCoverService`, `CycleFileLayout`.

`public void run(UUID jobId)`:
1. Load job, set `IN_PROGRESS`. Assert `job.getCycleId() != null`.
2. Load cycle → book → `book.title` + `book.description`
3. Call `aiTextProvider.generateImagePrompt(title, description)` → `prompt`
4. Call `aiImageProvider.generateImage(prompt)` → `byte[] imageBytes`
5. **Write base cover to disk:** `Path coverPath = cycleFileLayout.coverFile(cycleId)` → `Files.write(coverPath, imageBytes)`. This produces a real file that `VideoRenderJobRunner` can later read.
6. Encode to Base64, call `imagePayloadService.saveAiGeneratedImagePayload(cycleId, "ai", prompt, null, base64, aiImageProvider.mimeType(), coverPath.toString())` → `baseAsset` (creates the `BASE_AI_COVER` row with `storage_type = FILE_PATH` and `file_path` pointing to the cover file)
7. Count audio parts: `audioPartRepository.findByCycleId(cycleId).size()` → `count`
8. Build `BookCover` request with `cycleId`, `sourceImagePartId=baseAsset.getId()`, `title=book.title`, `count`, `imagePartType=cover_art`, `firstPartNumber=1`
9. Create temp output dir via `cycleFileLayout.tmpFile(cycleId, jobId, "covers")` parent
10. Call `bookCoverService.generateCovers(request, outputDir)` → check response status (creates per-part `COVER_ART` rows, each with inline or chunked base64)
11. Mark job `COMPLETED` (or `FAILED` on exception)

> **File-backed guarantee:** The base cover is always written to `{processing.temp.dir}/{cycleId}/covers/cover.png` AND stored as `image_asset.file_path`. The per-part `COVER_ART` rows are DB-only (base64) because `BookCoverService` deletes the temp PNGs after persisting. `VideoRenderJobRunner` resolves each part's cover by first checking `image_asset.file_path`; if null, it decodes `image_asset.inline_base64` to a temp file.

Wrap body in try/catch → on exception: set `errorMessage`, `FAILED`, save, rethrow.

- [ ] **Step 3: ImageGenerateJobRunnerTest** — Mockito unit test:
- `run_success`: mock all deps, verify `aiTextProvider.generateImagePrompt` called, verify `aiImageProvider.generateImage` called, verify `imagePayloadService.saveAiGeneratedImagePayload` called, verify job status = COMPLETED
- `run_aiFailure_marksJobFailed`: mock `aiTextProvider` to throw, verify job status = FAILED

- [ ] **Step 4: Commit**

```bash
./mvnw test -Dtest=ImageGenerateJobRunnerTest
git add -A && git commit -m "feat: ImageGenerateJobRunner — AI-driven cover generation pipeline"
```

---

## Task 7: VideoRenderJobRunner

**New:** `src/main/java/.../jobs/service/VideoRenderJobRunner.java`
**New:** `src/test/java/.../jobs/service/VideoRenderJobRunnerTest.java`

- [ ] **Step 1: VideoRenderJobRunner** — `@Component @RequiredArgsConstructor`:

> **Ownership:** VIDEO_RENDER owns a `video_part_id`. The orchestrator **pre-creates** one `video_part` row per `audio_part` (same `part_number`, status = `PENDING`) before enqueuing, and sets `job_queue.video_part_id` to the new row's ID. This means each VIDEO_RENDER job knows exactly which `video_part` it is responsible for.

> **Overlay resolution:** The runner reads `cycle.overlay_name` (added in Task 4 migration) — NOT from `job.inputPayload`. This keeps overlay selection a cycle-level setting.

Constructor-injected: `ProcessingJobRepository`, `AudioPartRepository`, `AudioStageRepository`, `ImageAssetRepository`, `VideoPartRepository`, `OverlayStorageService`, `VideoCreation`, `CycleFileLayout`, `CycleRepository`.

`public void run(UUID jobId)`:
1. Load job, set `IN_PROGRESS`. Assert `job.getVideoPartId() != null`.
2. Load `VideoPartEntity` by `job.videoPartId` → get `partNumber`, `cycleId`
3. Load matching `AudioPartEntity` via `audioPartRepository.findByCycleIdAndPartNumber(cycleId, partNumber)`
4. Find latest audio stage file: prefer COMPRESSED → CONCATENATED → TRIMMED (query `AudioStageRepository`)
5. Find cover image: `imageAssetRepository.findByCycleIdAndPartNumber(cycleId, partNumber)` → get file path or decode base64 to temp file
6. Resolve overlay: load `CycleEntity` → `cycle.getOverlayName()` → `overlayStorageService.resolve(overlayName)` → `particlesPath`
7. Compute output path: `cycleFileLayout.videoFile(cycleId, videoPart.getId())`
8. Call `videoCreation.start(coverPath, particlesPath, audioPath, outputPath, cycleId, partNumber, jobId)`
9. Mark job `COMPLETED`, update `videoPart.processingStatus = COMPLETED`

Wrap in try/catch → `FAILED` on exception, update `videoPart.processingStatus = FAILED`.

- [ ] **Step 2: VideoRenderJobRunnerTest** — Mockito unit test:
- `run_success`: mock deps, verify `videoCreation.start` called with correct paths, job COMPLETED
- `run_noAudioStage_fails`: no audio stage found, job FAILED

- [ ] **Step 3: Commit**

```bash
./mvnw test -Dtest=VideoRenderJobRunnerTest
git add -A && git commit -m "feat: VideoRenderJobRunner — video rendering via FFmpeg overlay + mux"
```

---

## Task 8: PlatformUploader strategy + DistributeJobRunner

**New files:**
- `src/main/java/.../poster/PlatformUploader.java`
- `src/main/java/.../poster/PlatformUploadRequest.java`
- `src/main/java/.../poster/PlatformUploadResult.java`
- `src/main/java/.../poster/BoostyPlatformUploader.java`
- `src/main/java/.../poster/YouTubePlatformUploader.java`
- `src/main/java/.../jobs/service/DistributeJobRunner.java`
- `src/test/java/.../jobs/service/DistributeJobRunnerTest.java`

- [ ] **Step 1: PlatformUploader interface**

```java
package kg.automation.rest.automatation.poster;

public interface PlatformUploader {
    String platformCode();  // "YOUTUBE", "BOOSTY", "TELEGRAM", etc.
    PlatformUploadResult upload(PlatformUploadRequest request);
}
```

- [ ] **Step 2: PlatformUploadRequest** — `@Builder @Getter`: `Long cycleId`, `String videoPath`, `String audioPath`, `String coverImagePath`, `String title`, `String description`, `int partNumber`

- [ ] **Step 3: PlatformUploadResult** — `@Builder @Getter`: `boolean success`, `String platformPostId`, `String url`, `String errorMessage`, `Map<String, Object> metadata`

- [ ] **Step 4: BoostyPlatformUploader** — `@Component`, wraps existing `Poster`. `platformCode()` returns `"BOOSTY"`. `upload()` calls `poster.postToBoosty(...)` (needs method visibility change or delegates to `poster.start()`). On success returns result with `success=true`. Catches exceptions → `success=false, errorMessage=...`.

- [ ] **Step 5: YouTubePlatformUploader** — `@Component`, wraps `YouTubeUploader`. `platformCode()` returns `"YOUTUBE"`. `upload()` catches `UnsupportedOperationException` → returns `success=false, errorMessage="YouTube upload not implemented"`.

- [ ] **Step 6: DistributeJobRunner** — `@Component @RequiredArgsConstructor`:

> **Ownership:** DISTRIBUTE owns a real `distribution_id`. The orchestrator **pre-creates** a `distribution` row (status = `PENDING`) representing the whole cycle's distribution batch, then sets `job_queue.distribution_id` to that row's ID. The runner loads this distribution row and writes per-platform `distribution_result` entries against it.

> **API change:** `DistributionPersistenceService.recordPlatformResult(...)` should accept `distributionId` (not `cycleId`) as the primary linkage, since the distribution batch row is the aggregate owner. The existing `resolveOrCreateDistribution` helper inside that service is no longer needed — the runner already has the `distributionId`. Add an overload or change the signature:
> ```java
> public void recordPlatformResult(
>         Long distributionId,        // ← primary linkage, not cycleId
>         String platformCode,
>         PlatformResultStatus status,
>         String platformPostId,
>         String url,
>         Map<String, Object> platformMetadata)
> ```
> Keep the old `cycleId`-based signature as `@Deprecated` for backward compatibility.

Constructor-injected: `ProcessingJobRepository`, `CycleRepository`, `DistributionRepository`, `DistributionPersistenceService`, `PlatformRepository`, `List<PlatformUploader> uploaders`.

`public void run(UUID jobId)`:
1. Load job, set `IN_PROGRESS`. Assert `job.getDistributionId() != null`.
2. Load `DistributionEntity` by `job.distributionId` → get `distributionId` and `cycleId`
3. Load cycle → parse `targetPlatforms` (comma-separated string → list)
4. Build uploader map: `uploaders.stream().collect(toMap(PlatformUploader::platformCode, identity()))`
5. Track counters: `successCount`, `failCount`
6. For each target platform code:
   a. Find uploader (if missing → log warn, skip)
   b. Build `PlatformUploadRequest` (resolve video/audio/cover paths from cycle data)
   c. Call `uploader.upload(request)` → `result`
   d. Call `distributionPersistenceService.recordPlatformResult(distributionId, platformCode, result.isSuccess() ? SUCCESS : FAILED, result.getPlatformPostId(), result.getUrl(), result.getMetadata())` — note: uses `distributionId`, not `cycleId`
   e. Increment counters
7. Update `distribution.status`: all success → `COMPLETED`, mixed → `PARTIAL`, all fail → `FAILED`. Save distribution.
8. Job is always marked `COMPLETED` (per-platform failures are isolated in `distribution_result`)

- [ ] **Step 7: DistributeJobRunnerTest** — Mockito unit test:
- `run_allPlatformsSucceed_distributionCompleted`: mock 2 uploaders both return success, verify `recordPlatformResult` called twice with SUCCESS
- `run_onePlatformFails_distributionPartial`: one uploader returns success, one fails → verify PARTIAL status
- `run_allFail_distributionFailed`: both fail → verify FAILED status, job still COMPLETED

- [ ] **Step 8: Commit**

```bash
./mvnw test -Dtest=DistributeJobRunnerTest
git add -A && git commit -m "feat: PlatformUploader strategy + DistributeJobRunner with partial-failure handling"
```

---

## Task 9: PipelineOrchestrator — advanceImageVideo + advanceDistribution

**Modified:** `src/main/java/.../pipeline/orchestrator/PipelineOrchestrator.java`
**Modified:** `src/main/java/.../pipeline/repository/VideoPartRepository.java` — add `List<VideoPartEntity> findByCycleId(Long cycleId)`
**New:** `src/test/java/.../pipeline/orchestrator/PipelineOrchestratorPostAudioTest.java`

New injected deps for orchestrator: `VideoPartRepository`, `DistributionRepository`, `CycleRepository`.

- [ ] **Step 1: Add `advanceImageVideo(long cycleId, UUID pipelineRunId)` method**

> **Idempotency contract:** Every transition is gated by `noJobsExist(cycleId, <next-type>, pipelineRunId)`. If `advance()` is called twice for the same state, the second call finds existing jobs and skips. Pre-created domain rows (`video_part`, `distribution`) use `findOrCreate` patterns so repeated calls never duplicate rows.

Phase transitions:

**AUDIO_COMPLETE → IMAGE_GENERATE:**
- Guard: `"AUDIO_COMPLETE".equals(run.getCurrentPhase()) && noJobsExist(cycleId, IMAGE_GENERATE, runId)`
- Action: create **one** `IMAGE_GENERATE` job with `cycleId` set, `imageAssetId = null` (cycle-owned; runner discovers/creates image assets). Set phase to `IMAGE_GENERATE`. Return `true`.

**IMAGE_GENERATE completed → VIDEO_RENDER:**
- Guard: `allJobsCompleted(cycleId, IMAGE_GENERATE, runId) && noJobsExist(cycleId, VIDEO_RENDER, runId)`
- Action: **idempotently pre-create** one `video_part` row per `audio_part` (uses `findByCycleIdAndPartNumber` — if row already exists, reuse it). Then create one `VIDEO_RENDER` job per `video_part`. Each job has `videoPartId` set. Set phase to `VIDEO_RENDER`. Return `true`.

```java
// Idempotent pre-creation of video_part rows
List<AudioPartEntity> parts = audioPartRepository.findByCycleId(cycleId);
for (AudioPartEntity ap : parts) {
    // findOrCreate — safe on repeated advance() calls
    VideoPartEntity vp = videoPartRepository
            .findByCycleIdAndPartNumber(cycleId, ap.getPartNumber())
            .orElseGet(() -> {
                VideoPartEntity v = new VideoPartEntity();
                v.setCycleId(cycleId);
                v.setPartNumber(ap.getPartNumber());
                v.setProcessingStatus(PartProcessingStatus.PENDING);
                return videoPartRepository.save(v);
            });
    createJob(JobType.VIDEO_RENDER, cycleId, ap.getId(), vp.getId(), null, null, pipelineRunId);
}
```

> **Why `noJobsExist` is sufficient:** The guard `noJobsExist(cycleId, VIDEO_RENDER, runId)` ensures that even if `video_part` rows already exist (from a previous partial attempt), the job creation block only runs once per pipeline run. The `findOrCreate` on `video_part` is a safety net for the domain row, not the job row.

**All VIDEO_RENDER completed → VIDEO_COMPLETE:**
- Guard: `allJobsCompleted(cycleId, VIDEO_RENDER, runId) && !"VIDEO_COMPLETE".equals(run.getCurrentPhase())`
- Action: set phase to `VIDEO_COMPLETE`. Return `true`.

- [ ] **Step 2: Add `advanceDistribution(long cycleId, UUID pipelineRunId)` method**

**VIDEO_COMPLETE → DISTRIBUTE:**
- Guard: `"VIDEO_COMPLETE".equals(run.getCurrentPhase()) && noJobsExist(cycleId, DISTRIBUTE, runId)`
- Action: **idempotently pre-create** a `distribution` row (uses `findByCycleIdAndAudioPartIdIsNull` — if row already exists, reuse it). Then create **one** `DISTRIBUTE` job with `distributionId` set. Set phase to `DISTRIBUTE`. Return `true`.

```java
// Idempotent pre-creation of distribution row
DistributionEntity dist = distributionRepository
        .findByCycleIdAndAudioPartIdIsNull(cycleId)
        .orElseGet(() -> {
            DistributionEntity d = new DistributionEntity();
            d.setCycleId(cycleId);
            d.setStatus(ReleaseStatus.PENDING);
            return distributionRepository.saveAndFlush(d);
        });
createJob(JobType.DISTRIBUTE, cycleId, null, null, null, dist.getId(), pipelineRunId);
```

**DISTRIBUTE completed → terminal state:**
- Guard: `allJobsCompleted(cycleId, DISTRIBUTE, runId)` and phase is not already terminal
- Action: read `distribution.status` to decide terminal state:
  - `distribution.status == COMPLETED` → phase = `PIPELINE_COMPLETE`, `pipeline_run.status = COMPLETED`
  - `distribution.status == PARTIAL` → phase = `PIPELINE_COMPLETE`, `pipeline_run.status = COMPLETED` (partial platform failures are acceptable; the pipeline itself finished)
  - `distribution.status == FAILED` → phase = `DISTRIBUTE_FAILED`, `pipeline_run.status = FAILED`

> **Design note:** PARTIAL distribution is not a pipeline failure — the pipeline ran to completion, but some platforms didn't succeed. The individual `distribution_result` rows carry the per-platform failure detail for retry. Only if ALL platforms fail does the pipeline_run itself fail.

> **Idempotency summary:** Repeated `advance()` calls are safe because:
> 1. `noJobsExist()` prevents duplicate `job_queue` rows for any given phase+run
> 2. `findOrCreate` patterns prevent duplicate `video_part` and `distribution` domain rows
> 3. Phase checks (e.g. `!"VIDEO_COMPLETE".equals(...)`) prevent re-executing terminal transitions

- [ ] **Step 3: Add top-level `advance(long cycleId, UUID pipelineRunId)` method**

Calls in sequence:
1. `advanceAudio(cycleId, pipelineRunId)` (existing)
2. `advanceImageVideo(cycleId, pipelineRunId)`
3. `advanceDistribution(cycleId, pipelineRunId)`

Each method returns `boolean` — `true` if it created new jobs (triggers early return so only one transition per call). The caller (listener) re-invokes `advance` after each job completion.

- [ ] **Step 4: Update `createJob` helper** to accept `distributionId` parameter (nullable). New signature:

```java
private ProcessingJobEntity createJob(
        JobType type, long cycleId, Long audioPartId,
        Long videoPartId, Long imageAssetId, Long distributionId, UUID pipelineRunId)
```

- [ ] **Step 5: PipelineOrchestratorPostAudioTest** — Mockito unit test:

Test cases:
- `advanceImageVideo_fromAudioComplete_createsImageGenerateJob`: pipeline at `AUDIO_COMPLETE` → verify one IMAGE_GENERATE job created with `cycleId` set and `imageAssetId = null`, phase = `IMAGE_GENERATE`
- `advanceImageVideo_imageComplete_preCreatesVideoPartsAndJobs`: IMAGE_GENERATE completed → verify N `video_part` rows created, N VIDEO_RENDER jobs created each with `videoPartId` set, phase = `VIDEO_RENDER`
- `advanceImageVideo_allVideosDone_setsVideoComplete`: all VIDEO_RENDER completed → phase = `VIDEO_COMPLETE`
- `advanceDistribution_fromVideoComplete_preCreatesDistributionAndJob`: VIDEO_COMPLETE → verify `distribution` row created, one DISTRIBUTE job with `distributionId` set, phase = `DISTRIBUTE`
- `advanceDistribution_distributeCompleted_distributionCompleted_pipelineComplete`: DISTRIBUTE completed + distribution.status=COMPLETED → `PIPELINE_COMPLETE`, run COMPLETED
- `advanceDistribution_distributeCompleted_distributionPartial_pipelineComplete`: DISTRIBUTE completed + distribution.status=PARTIAL → `PIPELINE_COMPLETE`, run COMPLETED
- `advanceDistribution_distributeCompleted_distributionFailed_pipelineFailed`: DISTRIBUTE completed + distribution.status=FAILED → `DISTRIBUTE_FAILED`, run FAILED
- `advance_fullChain_doesNotSkipPhases`: verify `advance()` at AUDIO_COMPLETE only triggers image, not video

- [ ] **Step 6: Commit**

```bash
./mvnw test -Dtest=PipelineOrchestratorPostAudioTest
git add -A && git commit -m "feat: PipelineOrchestrator — advanceImageVideo + advanceDistribution with pre-created FK rows"
```

---

## Task 10: AudioJobExecutor → PipelineJobExecutor + listener wiring

**Modified:**
- `src/main/java/.../jobs/runtime/AudioJobExecutor.java` — add IMAGE_GENERATE, VIDEO_RENDER, DISTRIBUTE cases
- `src/main/java/.../infra/rabbit/PipelineJobsRabbitListener.java` — dispatch new job types

- [ ] **Step 1: Extend AudioJobExecutor.enqueue()**

Add constructor-injected deps:
```java
private final ImageGenerateJobRunner imageGenerateJobRunner;
private final VideoRenderJobRunner videoRenderJobRunner;
private final DistributeJobRunner distributeJobRunner;
```

Extend the `switch` in `enqueue()`:
```java
case IMAGE_GENERATE -> imageGenerateJobRunner.run(jobId);
case VIDEO_RENDER   -> videoRenderJobRunner.run(jobId);
case DISTRIBUTE     -> distributeJobRunner.run(jobId);
```

- [ ] **Step 2: Extend PipelineJobsRabbitListener dispatch**

Add the new job types to the `onMessage` handler's switch/dispatch so IMAGE_GENERATE, VIDEO_RENDER, and DISTRIBUTE messages are routed to `audioJobExecutor.enqueue(jobId, jobType)`.

These job types go through the **standard priority queue** (not the noise-remove queue), so they use the existing `@RabbitListener(queues = QUEUE_PIPELINE_PRIORITY)` method.

- [ ] **Step 3: Call orchestrator after job completion**

In the listener's audit bridge (or in each runner's completion path), after a job finishes successfully, call `pipelineOrchestrator.advance(cycleId, pipelineRunId)` to trigger the next phase transition. This is the same pattern Plan A uses but now calls the top-level `advance()` instead of `advanceAudio()`.

- [ ] **Step 4: Update AutomatationApplicationTests mocks**

Add `@MockitoBean` for new runner beans:
```java
@MockitoBean ImageGenerateJobRunner imageGenerateJobRunner;
@MockitoBean VideoRenderJobRunner videoRenderJobRunner;
@MockitoBean DistributeJobRunner distributeJobRunner;
@MockitoBean OverlayStorageService overlayStorageService;
```

- [ ] **Step 5: Verify context loads + compile**

```bash
./mvnw test -Dtest=AutomatationApplicationTests
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: PipelineJobExecutor + listener dispatch for IMAGE_GENERATE, VIDEO_RENDER, DISTRIBUTE"
```

---

## Task 11: Final verification

- [ ] **Step 1: Run full test suite**

```bash
./mvnw test
```

Expected: BUILD SUCCESS — zero failures

- [ ] **Step 2: Verify Definition of Done checklist**

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | `./mvnw test` passes with zero failures | Step 1 above |
| 2 | `V202605080002` migration applies cleanly | `FlywayCycleSchemaIT` passes |
| 3 | `AiTextProvider` + `AiImageProvider` interfaces exist with at least one implementation each | `OllamaTextProvider` + `StubImageProvider` present |
| 4 | `OllamaTextProvider` has unit test with `MockRestServiceServer` | `OllamaTextProviderTest` passes |
| 5 | `OverlayController` accepts multipart upload and persists to `overlay_asset` | Compilation + manual verify |
| 6 | `ImageGenerateJobRunner` has tests (COMPLETED + FAILED paths) | `ImageGenerateJobRunnerTest` passes |
| 7 | `VideoRenderJobRunner` has tests (COMPLETED + FAILED paths) | `VideoRenderJobRunnerTest` passes |
| 8 | `DistributeJobRunner` has tests (all-success, partial, all-fail) | `DistributeJobRunnerTest` passes |
| 9 | `PipelineOrchestrator.advance()` transitions through IMAGE_GENERATE → VIDEO_RENDER → DISTRIBUTE → PIPELINE_COMPLETE | `PipelineOrchestratorPostAudioTest` passes |
| 10 | `PipelineJobsRabbitListener` dispatches IMAGE_GENERATE, VIDEO_RENDER, DISTRIBUTE | Code review of switch/dispatch |
| 11 | `AutomatationApplicationTests.contextLoads()` passes with all new mocks | Step 1 above |

- [ ] **Step 3: Commit tag**

```bash
git tag plan-b-complete
```

---

## Definition of done for Plan B

All of the following must be true:

1. `./mvnw test` — zero failures
2. Flyway migration `V202605080002` applies cleanly
3. `AiTextProvider` and `AiImageProvider` interfaces exist with config-driven selection
4. `OllamaTextProvider` has a unit test with `MockRestServiceServer`
5. `overlay_asset` table exists with REST upload endpoint
6. Three new job runners (`ImageGenerateJobRunner`, `VideoRenderJobRunner`, `DistributeJobRunner`) each have unit tests covering COMPLETED and FAILED paths
7. `DistributeJobRunner` handles partial failure (per-platform `distribution_result` isolation)
8. `PipelineOrchestrator.advance()` handles full pipeline: AUDIO_COMPLETE → IMAGE_GENERATE → VIDEO_RENDER → DISTRIBUTE → PIPELINE_COMPLETE
9. `PipelineJobsRabbitListener` dispatches all new job types
10. `AutomatationApplicationTests.contextLoads()` passes

**Continue with Plan C** for cleanup lifecycle, retry infrastructure, and monitoring dashboard.

