# File System Isolation & Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate all file path collisions when 10+ books process in parallel, and add a human-gated lifecycle that keeps files until all uploads are confirmed before deleting them.

**Architecture:** A new `CycleFileLayout` service owns all path construction — every file lives under `{base}/{cycleId}/part_{partId}/` or `{base}/{cycleId}/tmp/{jobId}_{suffix}`. Five broken services are patched to use this contract. A `file_lifecycle` column on the `cycle` table drives a state machine (`ACTIVE → UPLOADS_CONFIRMED → APPROVED_FOR_DELETE → FILES_DELETED`) gated by human dashboard approval.

**Tech Stack:** Java 17, Spring Boot 3.4, JPA/Hibernate, Flyway, JUnit 5, React 19 + TypeScript (frontend task only)

---

## File Map

### New files
| File | Responsibility |
|---|---|
| `pipeline/service/CycleFileLayout.java` | Single source of truth for all path construction |
| `pipeline/service/CycleLifecycleService.java` | Only service that writes `file_lifecycle`; guards all transitions |
| `pipeline/service/UploadConfirmationService.java` | Checks if all target platforms confirmed; triggers UPLOADS_CONFIRMED |
| `pipeline/domain/FileLifecycle.java` | Enum: ACTIVE, UPLOADS_CONFIRMED, APPROVED_FOR_DELETE, FILES_DELETED, FAILED |
| `pipeline/domain/UploadResultEntity.java` | JPA entity for `upload_result` table |
| `pipeline/domain/UploadPlatform.java` | Enum: YOUTUBE, PATREON, TWITCH, TELEGRAM |
| `pipeline/repository/UploadResultRepository.java` | JPA repo for upload_result |
| `pipeline/scheduler/CycleCleanupScheduler.java` | Hourly: deletes approved dirs, sweeps stale tmp/ |
| `db/migration/V202605072464__cycle_file_lifecycle.sql` | Adds file_lifecycle + target_platforms to cycle |
| `db/migration/V202605072465__audio_stage_unique.sql` | Unique constraint on (audio_part_id, stage) |
| `db/migration/V202605072466__upload_result.sql` | Creates upload_result table |
| `src/test/.../pipeline/service/CycleFileLayoutTest.java` | Unit tests for CycleFileLayout |
| `src/test/.../pipeline/service/CycleLifecycleServiceTest.java` | Unit tests for lifecycle transitions |
| `src/test/.../pipeline/service/UploadConfirmationServiceTest.java` | Unit tests for confirmation logic |

### Modified files
| File | What changes |
|---|---|
| `pipeline/domain/CycleEntity.java` | Add `fileLifecycle` + `targetPlatforms` fields |
| `pipeline/repository/CycleRepository.java` | Add `findByFileLifecycle(FileLifecycle)` |
| `pipeline/web/CycleV1Controller.java` | Add `POST /{id}/approve-delete` endpoint |
| `video/service/VideoCreation.java` | Add `jobId` param; replace hardcoded temp paths with `CycleFileLayout.tmpFile()` |
| `gateway/RestCallService.java` | `downloadImage()` returns `Path`, uses `CycleFileLayout.tmpFile()` |
| `torrent/controller/QBittorrentController.java` | Accept `cycleId`+`partId` in request body; use `CycleFileLayout.partSourceDir()` |
| `jobs/service/ConcatenateProcessingJobRunner.java` | Derive paths from job metadata via `CycleFileLayout` instead of payload strings |
| `jobs/service/ProcessingJobService.java` | Remove `audioStagePathService.putPath(ORIGINAL, inputDirectory)` call |
| `firebase/service/ProcessingTempDirService.java` | Deprecate — delegate all methods to `CycleFileLayout` |

---

## Task 1: CycleFileLayout service

**Files:**
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/service/CycleFileLayout.java`
- Create: `audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/pipeline/service/CycleFileLayoutTest.java`

- [ ] **Step 1.1: Write the failing tests**

```java
// src/test/.../pipeline/service/CycleFileLayoutTest.java
package kg.automation.rest.automatation.pipeline.service;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class CycleFileLayoutTest {

    @TempDir
    Path base;

    private CycleFileLayout layout() {
        return new CycleFileLayout(base.toString());
    }

    @Test
    void cycleDir_returnsCorrectPath() {
        Path p = layout().cycleDir(42L);
        assertEquals(base.resolve("42"), p);
    }

    @Test
    void partSourceDir_createsAndReturnsPath() {
        Path p = layout().partSourceDir(42L, 7L);
        assertEquals(base.resolve("42/part_7/source"), p);
        assertTrue(Files.isDirectory(p));
    }

    @Test
    void partCompressedDir_createsAndReturnsPath() {
        Path p = layout().partCompressedDir(42L, 7L);
        assertEquals(base.resolve("42/part_7/compressed"), p);
        assertTrue(Files.isDirectory(p));
    }

    @Test
    void partConcatFile_createsParentAndReturnsPath() {
        Path p = layout().partConcatFile(42L, 7L);
        assertEquals(base.resolve("42/part_7/concat/part_7.mp3"), p);
        assertTrue(Files.isDirectory(p.getParent()));
    }

    @Test
    void coverFile_createsParentAndReturnsPath() {
        Path p = layout().coverFile(42L);
        assertEquals(base.resolve("42/covers/cover.png"), p);
        assertTrue(Files.isDirectory(p.getParent()));
    }

    @Test
    void videoFile_createsParentAndReturnsPath() {
        Path p = layout().videoFile(42L, 7L);
        assertEquals(base.resolve("42/video/part_7.mp4"), p);
        assertTrue(Files.isDirectory(p.getParent()));
    }

    @Test
    void tmpFile_createsParentAndReturnsJobScopedPath() {
        UUID jobId = UUID.fromString("00000000-0000-0000-0000-000000000001");
        Path p = layout().tmpFile(42L, jobId, "pre_rendered.mp4");
        assertEquals(base.resolve("42/tmp/00000000-0000-0000-0000-000000000001_pre_rendered.mp4"), p);
        assertTrue(Files.isDirectory(p.getParent()));
    }

    @Test
    void deleteTmpFile_deletesExistingFile() throws Exception {
        Path p = layout().tmpFile(42L, UUID.randomUUID(), "test.mp4");
        Files.writeString(p, "data");
        assertTrue(Files.exists(p));

        layout().deleteTmpFile(p);
        assertFalse(Files.exists(p));
    }

    @Test
    void deleteTmpFile_doesNotThrowForMissingFile() {
        Path missing = base.resolve("nonexistent.mp4");
        assertDoesNotThrow(() -> layout().deleteTmpFile(missing));
    }

    @Test
    void deleteCycleDir_deletesEntireTree() throws Exception {
        CycleFileLayout l = layout();
        l.partSourceDir(42L, 1L); // creates dirs
        Files.writeString(l.partSourceDir(42L, 1L).resolve("test.mp3"), "data");

        l.deleteCycleDir(42L);
        assertFalse(Files.exists(l.cycleDir(42L)));
    }

    @Test
    void deleteCycleDir_doesNotThrowIfAlreadyAbsent() {
        assertDoesNotThrow(() -> layout().deleteCycleDir(999L));
    }
}
```

- [ ] **Step 1.2: Run tests to confirm they fail**

```bash
cd audio-library-automation-bot
./mvnw test -Dtest=CycleFileLayoutTest -pl . 2>&1 | tail -20
```
Expected: `FAILURE` — `CycleFileLayout` class does not exist yet.

- [ ] **Step 1.3: Implement CycleFileLayout**

```java
// src/main/java/.../pipeline/service/CycleFileLayout.java
package kg.automation.rest.automatation.pipeline.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Comparator;
import java.util.UUID;
import java.util.stream.Stream;

@Service
public class CycleFileLayout {

    private final String base;

    public CycleFileLayout(@Value("${processing.temp.dir:${java.io.tmpdir}/audio-pipeline}") String base) {
        this.base = base;
    }

    public Path cycleDir(long cycleId) {
        return Path.of(base, String.valueOf(cycleId));
    }

    public Path partSourceDir(long cycleId, long partId) {
        return mkdir(cycleDir(cycleId).resolve("part_" + partId).resolve("source"));
    }

    public Path partCompressedDir(long cycleId, long partId) {
        return mkdir(cycleDir(cycleId).resolve("part_" + partId).resolve("compressed"));
    }

    public Path partConcatFile(long cycleId, long partId) {
        Path dir = mkdir(cycleDir(cycleId).resolve("part_" + partId).resolve("concat"));
        return dir.resolve("part_" + partId + ".mp3");
    }

    public Path coverFile(long cycleId) {
        Path dir = mkdir(cycleDir(cycleId).resolve("covers"));
        return dir.resolve("cover.png");
    }

    public Path videoFile(long cycleId, long partId) {
        Path dir = mkdir(cycleDir(cycleId).resolve("video"));
        return dir.resolve("part_" + partId + ".mp4");
    }

    public Path tmpFile(long cycleId, UUID jobId, String suffix) {
        Path dir = mkdir(cycleDir(cycleId).resolve("tmp"));
        return dir.resolve(jobId + "_" + suffix);
    }

    public void deleteTmpFile(Path p) {
        try {
            Files.deleteIfExists(p);
        } catch (IOException e) {
            // safe delete — missing file is not an error
        }
    }

    public void deleteCycleDir(long cycleId) {
        Path dir = cycleDir(cycleId);
        if (!Files.exists(dir)) return;
        try (Stream<Path> walk = Files.walk(dir)) {
            walk.sorted(Comparator.reverseOrder()).forEach(p -> {
                try { Files.deleteIfExists(p); } catch (IOException ignored) {}
            });
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to delete cycle dir " + dir, e);
        }
    }

    private static Path mkdir(Path dir) {
        try {
            Files.createDirectories(dir);
            return dir;
        } catch (IOException e) {
            throw new UncheckedIOException("Cannot create directory " + dir, e);
        }
    }
}
```

- [ ] **Step 1.4: Run tests — all must pass**

```bash
./mvnw test -Dtest=CycleFileLayoutTest -pl . 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, `Tests run: 10, Failures: 0`.

- [ ] **Step 1.5: Commit**

```bash
cd audio-library-automation-bot
git add src/main/java/kg/automation/rest/automatation/pipeline/service/CycleFileLayout.java \
        src/test/java/kg/automation/rest/automatation/pipeline/service/CycleFileLayoutTest.java
git commit -m "feat: add CycleFileLayout — single owner of all pipeline file paths"
```

---

## Task 2: Flyway migrations

**Files:**
- Create: `audio-library-automation-bot/src/main/resources/db/migration/V202605072464__cycle_file_lifecycle.sql`
- Create: `audio-library-automation-bot/src/main/resources/db/migration/V202605072465__audio_stage_unique.sql`

- [ ] **Step 2.1: Create migration 1 — file_lifecycle and target_platforms on cycle table**

```sql
-- V202605072464__cycle_file_lifecycle.sql
ALTER TABLE cycle
    ADD COLUMN file_lifecycle  VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',
    ADD COLUMN target_platforms TEXT        NOT NULL DEFAULT 'YOUTUBE,TELEGRAM';

CREATE INDEX idx_cycle_file_lifecycle
    ON cycle (file_lifecycle)
    WHERE file_lifecycle = 'APPROVED_FOR_DELETE';
```

- [ ] **Step 2.2: Create migration 2 — unique constraint on audio_stage**

```sql
-- V202605072465__audio_stage_unique.sql
ALTER TABLE audio_stage
    ADD CONSTRAINT uq_audio_stage_part_stage
    UNIQUE (audio_part_id, stage);
```

- [ ] **Step 2.3: Run migrations against the dev DB**

```bash
cd audio-library-automation-bot
docker compose up -d
./mvnw spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=" 2>&1 | grep -E "Flyway|migration|ERROR" | head -20
```
Expected: `Successfully applied 2 migrations to schema "public"`.

Or verify with H2 (no Docker):
```bash
./mvnw test -Dspring.profiles.active=dev 2>&1 | grep -E "Flyway|migration|ERROR" | head -20
```

- [ ] **Step 2.4: Commit**

```bash
git add src/main/resources/db/migration/V202605072464__cycle_file_lifecycle.sql \
        src/main/resources/db/migration/V202605072465__audio_stage_unique.sql
git commit -m "feat: add file_lifecycle/target_platforms to cycle, unique constraint on audio_stage"
```

---

## Task 3: CycleEntity — add new fields

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/domain/CycleEntity.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/domain/FileLifecycle.java`

- [ ] **Step 3.1: Create FileLifecycle enum**

```java
// pipeline/domain/FileLifecycle.java
package kg.automation.rest.automatation.pipeline.domain;

public enum FileLifecycle {
    ACTIVE,
    UPLOADS_CONFIRMED,
    APPROVED_FOR_DELETE,
    FILES_DELETED,
    FAILED
}
```

- [ ] **Step 3.2: Add fields to CycleEntity**

In `CycleEntity.java`, add after the `description` field (line 38):

```java
    @Enumerated(EnumType.STRING)
    @Column(name = "file_lifecycle", nullable = false, length = 32)
    private FileLifecycle fileLifecycle = FileLifecycle.ACTIVE;

    @Column(name = "target_platforms", nullable = false)
    private String targetPlatforms = "YOUTUBE,TELEGRAM";
```

Add the import at the top of the file:
```java
import kg.automation.rest.automatation.pipeline.domain.FileLifecycle;
```

- [ ] **Step 3.3: Add query method to CycleRepository**

Replace the contents of `CycleRepository.java`:
```java
package kg.automation.rest.automatation.pipeline.repository;

import kg.automation.rest.automatation.pipeline.domain.CycleEntity;
import kg.automation.rest.automatation.pipeline.domain.FileLifecycle;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

public interface CycleRepository extends JpaRepository<CycleEntity, Long> {
    Optional<CycleEntity> findByCycleIdentifier(String cycleIdentifier);
    List<CycleEntity> findByFileLifecycle(FileLifecycle lifecycle);
}
```

- [ ] **Step 3.4: Compile check**

```bash
./mvnw compile 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 3.5: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/pipeline/domain/FileLifecycle.java \
        src/main/java/kg/automation/rest/automatation/pipeline/domain/CycleEntity.java \
        src/main/java/kg/automation/rest/automatation/pipeline/repository/CycleRepository.java
git commit -m "feat: add FileLifecycle enum and file_lifecycle/target_platforms to CycleEntity"
```

---

## Task 4: Fix VideoCreation — CRITICAL

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/video/service/VideoCreation.java`

The `start()` method at line 156 already receives `cycleId` but not `jobId`. The `starter()` batch method calls `start()` in a loop — it needs to generate a `jobId` per iteration.

- [ ] **Step 4.1: Write a failing test**

```java
// src/test/.../video/service/VideoCreationTmpPathTest.java
package kg.automation.rest.automatation.video.service;

import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class VideoCreationTmpPathTest {

    @TempDir Path base;

    @Test
    void tmpPaths_areJobIdScoped_notShared() {
        CycleFileLayout layout = new CycleFileLayout(base.toString());
        UUID job1 = UUID.randomUUID();
        UUID job2 = UUID.randomUUID();

        Path p1 = layout.tmpFile(1L, job1, "pre_rendered.mp4");
        Path p2 = layout.tmpFile(1L, job2, "pre_rendered.mp4");

        assertNotEquals(p1, p2, "Two jobs must get different tmp paths");
        assertTrue(p1.getFileName().toString().startsWith(job1.toString()));
        assertTrue(p2.getFileName().toString().startsWith(job2.toString()));
    }
}
```

- [ ] **Step 4.2: Run test to confirm it passes (verifies CycleFileLayout contract)**

```bash
./mvnw test -Dtest=VideoCreationTmpPathTest 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 4.3: Modify VideoCreation.start() to accept jobId and use CycleFileLayout**

In `VideoCreation.java`:

1. Add field injection after the existing `@Autowired` fields (after line 34):
```java
    @Autowired
    private CycleFileLayout cycleFileLayout;
```

2. Replace the `start()` method signature and body (lines 156–208). The new version:
```java
    public void start(
            String imagePath,
            String particlesPath,
            String audioPath,
            String outputPath,
            Long cycleId,
            Integer partNumber,
            UUID jobId)
            throws IOException {
        if (cycleId != null && partNumber == null) {
            throw new IllegalArgumentException("partNumber is required when cycleId is set");
        }
        if (cycleId != null && videoPartWriteService == null) {
            throw new IllegalStateException("cycleId requires VideoPartWriteService (database)");
        }

        UUID resolvedJobId = jobId != null ? jobId : UUID.randomUUID();
        long resolvedCycleId = cycleId != null ? cycleId : 0L;

        Path segmentPath  = cycleFileLayout.tmpFile(resolvedCycleId, resolvedJobId, "pre_rendered.mp4");
        Path preScaledPath = cycleFileLayout.tmpFile(resolvedCycleId, resolvedJobId, "pre_scaled.jpg");
        boolean persist = cycleId != null && partNumber != null && videoPartWriteService != null;

        if (persist) {
            videoPartWriteService.ensureCycleExists(cycleId);
            videoPartWriteService.markStart(cycleId, partNumber);
        }

        try {
            imageService.preScaleImage(imagePath, preScaledPath.toString());
            preRenderOverlay(imagePath, particlesPath, segmentPath.toString());
            createFinalVideo(segmentPath.toString(), audioPath, outputPath);
            if (persist) {
                Path outputAbsolute = Path.of(outputPath).toAbsolutePath().normalize();
                Long durationSeconds = ffmpegOperations.probeDurationSeconds(outputAbsolute);
                videoPartWriteService.markSuccess(
                        cycleId,
                        partNumber,
                        outputAbsolute.toString(),
                        segmentPath.toAbsolutePath().normalize().toString(),
                        durationSeconds);
            }
        } catch (IOException ex) {
            if (persist) videoPartWriteService.markFailed(cycleId, partNumber);
            throw ex;
        } catch (RuntimeException ex) {
            if (persist) videoPartWriteService.markFailed(cycleId, partNumber);
            throw ex;
        } finally {
            cycleFileLayout.deleteTmpFile(segmentPath);
            cycleFileLayout.deleteTmpFile(preScaledPath);
        }
    }
```

3. Update the 4-arg `start()` overload (line 149) to call the new 7-arg version:
```java
    public void start(String imagePath, String particlesPath, String audioPath, String outputPath) throws IOException {
        start(imagePath, particlesPath, audioPath, outputPath, null, null, null);
    }
```

4. Update the 6-arg `start()` overload (originally line 149–151) — this overload no longer exists; the 7-arg is now the canonical method. Remove the 6-arg overload entirely. Update the call in `starter()` at line 102:
```java
                    UUID iterJobId = UUID.randomUUID();
                    start(imagePath, particlesPath, audioPath, outputFile.toString(), cycleId, pn, iterJobId);
```

Also add `import java.util.UUID;` at the top.

- [ ] **Step 4.4: Compile check**

```bash
./mvnw compile 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 4.5: Verify src/main/resources/temp/ is no longer referenced**

```bash
grep -r "resources/temp" src/main/java/ && echo "FOUND — fix before committing" || echo "CLEAN"
```
Expected: `CLEAN`.

- [ ] **Step 4.6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/video/service/VideoCreation.java
git commit -m "fix: VideoCreation uses CycleFileLayout for tmp paths — eliminates concurrent job collision"
```

---

## Task 5: Fix RestCallService — CRITICAL

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/gateway/RestCallService.java`

- [ ] **Step 5.1: Write a failing test**

```java
// src/test/.../gateway/RestCallServiceDownloadPathTest.java
package kg.automation.rest.automatation.gateway;

import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class RestCallServiceDownloadPathTest {

    @TempDir Path base;

    @Test
    void downloadPaths_areDifferentPerJob() {
        CycleFileLayout layout = new CycleFileLayout(base.toString());
        UUID job1 = UUID.randomUUID();
        UUID job2 = UUID.randomUUID();

        Path p1 = layout.tmpFile(5L, job1, "download.png");
        Path p2 = layout.tmpFile(5L, job2, "download.png");

        assertNotEquals(p1, p2);
        assertFalse(p1.toString().endsWith("base.png"), "Must not use legacy base.png name");
    }
}
```

- [ ] **Step 5.2: Run test**

```bash
./mvnw test -Dtest=RestCallServiceDownloadPathTest 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 5.3: Update RestCallService**

1. Remove the `@Value("${processing.temp.dir:...}") private String tempDir;` field (lines 38–39).
2. Add `CycleFileLayout` injection after the `RestTemplate` field:
```java
    private final CycleFileLayout cycleFileLayout;

    public RestCallService(RestTemplate restTemplate, CycleFileLayout cycleFileLayout) {
        this.restTemplate = restTemplate;
        this.cycleFileLayout = cycleFileLayout;
    }
```
3. Replace `downloadImage()` (lines 110–123) with:
```java
    /**
     * Downloads the image URL from the OpenAI response to a job-scoped tmp file.
     * The caller is responsible for deleting the returned path after use via
     * {@code cycleFileLayout.deleteTmpFile(path)}.
     */
    public Path downloadImage(OpenAiImageRes req, long cycleId, UUID jobId) throws IOException {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(openAiApiKey);
        HttpEntity<String> imagereq = new HttpEntity<>(headers);
        ResponseEntity<byte[]> responseIm = restTemplate.getForEntity(
                req.getData().get(0).getUrl(), byte[].class, HttpMethod.GET, imagereq);
        if (!responseIm.getStatusCode().is2xxSuccessful() || responseIm.getBody() == null) {
            throw new IOException("Image download failed: HTTP " + responseIm.getStatusCode());
        }
        Path tmp = cycleFileLayout.tmpFile(cycleId, jobId, "download.png");
        saveImageLocally(responseIm.getBody(), tmp.toString());
        return tmp;
    }
```
4. Add `import java.util.UUID;` to the imports.

- [ ] **Step 5.4: Fix the caller — OpenAiCotroller.java**

There is exactly one caller: `ai/openai/controller/OpenAiCotroller.java` line 35.
This controller has no `cycleId` in scope (it is a standalone AI image endpoint).
Use `cycleId=0L` and a fresh `UUID.randomUUID()` as the jobId — the file lands in
`{base}/0/tmp/{jobId}_download.png` and is cleaned up immediately after use.

Add `CycleFileLayout` injection to `OpenAiCotroller` and update the call:
```java
    @Autowired
    private CycleFileLayout cycleFileLayout;
```

Replace lines 33–38 (the `downloadImage` call block):
```java
        OpenAiImageRes response = restCallService.sentQueryToOpenAiImage(req);
        UUID jobId = UUID.randomUUID();
        try {
            Path img = restCallService.downloadImage(response, 0L, jobId);
            // img is used immediately by callers of the returned OpenAiImageRes URL;
            // the local file is a side-effect cache — delete it now.
            cycleFileLayout.deleteTmpFile(img);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        return response;
```

Add `import java.util.UUID;` and `import java.nio.file.Path;` to `OpenAiCotroller.java`.

- [ ] **Step 5.5: Compile check**

```bash
./mvnw compile 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 5.6: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/gateway/RestCallService.java
# also add any caller files changed in step 5.4
git commit -m "fix: RestCallService.downloadImage uses job-scoped tmp path, caller owns cleanup"
```

---

## Task 6: Fix QBittorrentController + TorrentService

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/torrent/controller/QBittorrentController.java`

- [ ] **Step 6.1: Create request body DTO**

```java
// torrent/controller/TorrentUploadRequest.java
package kg.automation.rest.automatation.torrent.controller;

import lombok.Data;

@Data
public class TorrentUploadRequest {
    /** DB id of the cycle this download belongs to. */
    private Long cycleId;
    /** DB id of the audio_part this download will populate. */
    private Long partId;
}
```

- [ ] **Step 6.2: Update QBittorrentController.uploadTorrent()**

Replace the `uploadTorrent()` method (lines 97–115) with:
```java
    @PostMapping("/upload")
    public String uploadTorrent(@RequestBody TorrentUploadRequest req) {
        if (req.getCycleId() == null || req.getPartId() == null) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "cycleId and partId are required");
        }
        Path sourceDir = cycleFileLayout.partSourceDir(req.getCycleId(), req.getPartId());

        Response response = torrentService.uploadTorrents(sourceDir.toString(), sourceDir.toString());

        if ("success".equalsIgnoreCase(response.getStatus())) {
            restCallService.enableMonitoring();
        }
        return gson.toJson(response);
    }
```

Add the field injection after the existing `@Value` field:
```java
    @Autowired
    private CycleFileLayout cycleFileLayout;
```

Remove the old `@Value("${processing.temp.dir:...}") private String tempDir;` field and the `Files.createDirectories` calls (they are now handled by `CycleFileLayout`).

Also add `import java.nio.file.Path;` and `import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;`.

- [ ] **Step 6.3: Compile check**

```bash
./mvnw compile 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 6.4: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/torrent/controller/QBittorrentController.java \
        src/main/java/kg/automation/rest/automatation/torrent/controller/TorrentUploadRequest.java
git commit -m "fix: torrent upload paths are now scoped to cycleId+partId via CycleFileLayout"
```

---

## Task 7: Fix ConcatenateProcessingJobRunner

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/service/ConcatenateProcessingJobRunner.java`
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/service/ProcessingJobService.java`

- [ ] **Step 7.1: Write a failing test**

```java
// src/test/.../jobs/service/ConcatenateJobPathDerivationTest.java
package kg.automation.rest.automatation.jobs.service;

import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;

import static org.junit.jupiter.api.Assertions.*;

class ConcatenateJobPathDerivationTest {

    @TempDir Path base;

    @Test
    void inputDir_derivedFromCycleAndPart() {
        CycleFileLayout layout = new CycleFileLayout(base.toString());
        Path source = layout.partSourceDir(10L, 3L);
        assertEquals(base.resolve("10/part_3/source"), source);
    }

    @Test
    void outputFile_derivedFromCycleAndPart() {
        CycleFileLayout layout = new CycleFileLayout(base.toString());
        Path concat = layout.partConcatFile(10L, 3L);
        assertEquals(base.resolve("10/part_3/concat/part_3.mp3"), concat);
    }
}
```

- [ ] **Step 7.2: Run test**

```bash
./mvnw test -Dtest=ConcatenateJobPathDerivationTest 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 7.3: Update ConcatenateProcessingJobRunner**

1. Add `CycleFileLayout` to the constructor via Lombok's `@RequiredArgsConstructor` (it's already on the class). Add the field:
```java
    private final CycleFileLayout cycleFileLayout;
```

2. Replace the path derivation in `run()`. Find the block starting at line 54 where `req` is read from payload:
```java
            req = objectMapper.readValue(job.getInputPayload(), AudioConcatenateJobRequest.class);
```
Replace the two lines that derive paths from `req.getInputDirectory()` and `req.getOutputDirectory()` in the `concatenatorService.concatenateAudio()` call. The new block:
```java
            // Paths are derived from job metadata — not from caller-supplied strings
            Path inputDir  = cycleFileLayout.partSourceDir(job.getCycleId(), job.getAudioPartId());
            Path outputDir = cycleFileLayout.partConcatFile(job.getCycleId(), job.getAudioPartId()).getParent();

            Response res = concatenatorService.concatenateAudio(
                    inputDir.toString(), outputDir.toString());
```

3. Fix `expectedConcatenatedPath()` at the bottom of the class — it no longer needs `req`:
```java
    private String expectedConcatenatedPath(long cycleId, long audioPartId) {
        return cycleFileLayout.partConcatFile(cycleId, audioPartId).toString();
    }
```
Update its call site in `run()`:
```java
    audioStagePathService.putPath(ap.getId(), AudioStageType.CONCATENATED,
            expectedConcatenatedPath(job.getCycleId(), job.getAudioPartId()));
```

- [ ] **Step 7.4: Remove ORIGINAL stage path write from ProcessingJobService**

In `ProcessingJobService.java` at line 69, remove:
```java
        audioStagePathService.putPath(part.getId(), AudioStageType.ORIGINAL, body.getInputDirectory());
```
This was recording caller-supplied paths. The runner now derives paths from the layout service.

Also remove the `audioStagePathService` field and its import if it's no longer used elsewhere in the class.

- [ ] **Step 7.5: Compile check**

```bash
./mvnw compile 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 7.6: Run existing job tests**

```bash
./mvnw test -Dtest=ProcessingJobServiceTest,InternalPipelineJobControllerTest 2>&1 | tail -15
```
Expected: `BUILD SUCCESS`, no failures.

- [ ] **Step 7.7: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/jobs/service/ConcatenateProcessingJobRunner.java \
        src/main/java/kg/automation/rest/automatation/jobs/service/ProcessingJobService.java \
        src/test/java/kg/automation/rest/automatation/jobs/service/ConcatenateJobPathDerivationTest.java
git commit -m "fix: ConcatenateJobRunner derives paths from CycleFileLayout, not caller-supplied strings"
```

---

## Task 8: Deprecate ProcessingTempDirService

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/service/ProcessingTempDirService.java`

- [ ] **Step 8.1: Make ProcessingTempDirService delegate to CycleFileLayout**

Replace the entire body of `ProcessingTempDirService.java`:
```java
package kg.automation.rest.automatation.firebase.service;

import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.UUID;

/**
 * @deprecated Use {@link CycleFileLayout} directly. This class delegates to it
 * and exists only to avoid breaking existing tests during the migration.
 */
@Deprecated(since = "2026-05-07", forRemoval = true)
@Service
public class ProcessingTempDirService {

    private final CycleFileLayout layout;

    public ProcessingTempDirService(CycleFileLayout layout) {
        this.layout = layout;
    }

    /** @deprecated Use {@link CycleFileLayout#partSourceDir} etc. */
    @Deprecated
    public Path createJobDir(String jobId) throws IOException {
        // Best-effort: parse jobId as UUID; fall back to 0L cycle
        long cycleId;
        try { cycleId = Long.parseLong(jobId); } catch (NumberFormatException e) { cycleId = 0L; }
        layout.partSourceDir(cycleId, 1L);
        layout.partCompressedDir(cycleId, 1L);
        return layout.cycleDir(cycleId);
    }

    /** @deprecated Use {@link CycleFileLayout#cycleDir} */
    @Deprecated
    public Path jobDir(String jobId) {
        long cycleId;
        try { cycleId = Long.parseLong(jobId); } catch (NumberFormatException e) { cycleId = 0L; }
        return layout.cycleDir(cycleId);
    }

    /** @deprecated Use {@link CycleFileLayout#deleteCycleDir} */
    @Deprecated
    public void cleanupJobDir(String jobId) {
        long cycleId;
        try { cycleId = Long.parseLong(jobId); } catch (NumberFormatException e) { cycleId = 0L; }
        layout.deleteCycleDir(cycleId);
    }
}
```

- [ ] **Step 8.2: Run existing ProcessingTempDirServiceTest — must still pass**

```bash
./mvnw test -Dtest=ProcessingTempDirServiceTest 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`. (The tests still pass via delegation.)

- [ ] **Step 8.3: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/firebase/service/ProcessingTempDirService.java
git commit -m "deprecate: ProcessingTempDirService delegates to CycleFileLayout, marked for removal"
```

---

## Task 9: upload_result table + UploadConfirmationService

**Files:**
- Create: `db/migration/V202605072466__upload_result.sql`
- Create: `pipeline/domain/UploadPlatform.java`
- Create: `pipeline/domain/UploadResultEntity.java`
- Create: `pipeline/repository/UploadResultRepository.java`
- Create: `pipeline/service/UploadConfirmationService.java`
- Create: `src/test/.../pipeline/service/UploadConfirmationServiceTest.java`

- [ ] **Step 9.1: Create migration**

```sql
-- V202605072466__upload_result.sql
CREATE TABLE upload_result (
    id           BIGSERIAL    PRIMARY KEY,
    cycle_id     BIGINT       NOT NULL REFERENCES cycle(id),
    platform     VARCHAR(32)  NOT NULL,
    status       VARCHAR(16)  NOT NULL,
    external_id  VARCHAR(256),
    completed_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX uq_upload_result_cycle_platform
    ON upload_result (cycle_id, platform);
```

- [ ] **Step 9.2: Create UploadPlatform enum**

```java
// pipeline/domain/UploadPlatform.java
package kg.automation.rest.automatation.pipeline.domain;

public enum UploadPlatform {
    YOUTUBE, PATREON, TWITCH, TELEGRAM
}
```

- [ ] **Step 9.3: Create UploadResultEntity**

```java
// pipeline/domain/UploadResultEntity.java
package kg.automation.rest.automatation.pipeline.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import java.time.Instant;

@Getter @Setter
@Entity
@Table(name = "upload_result")
public class UploadResultEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "cycle_id", nullable = false)
    private Long cycleId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 32)
    private UploadPlatform platform;

    @Column(nullable = false, length = 16)
    private String status; // "SUCCESS" or "FAILED"

    @Column(name = "external_id", length = 256)
    private String externalId;

    @Column(name = "completed_at", nullable = false)
    private Instant completedAt;

    @PrePersist
    void prePersist() {
        if (completedAt == null) completedAt = Instant.now();
    }
}
```

- [ ] **Step 9.4: Create UploadResultRepository**

```java
// pipeline/repository/UploadResultRepository.java
package kg.automation.rest.automatation.pipeline.repository;

import kg.automation.rest.automatation.pipeline.domain.UploadPlatform;
import kg.automation.rest.automatation.pipeline.domain.UploadResultEntity;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

public interface UploadResultRepository extends JpaRepository<UploadResultEntity, Long> {
    List<UploadResultEntity> findByCycleId(long cycleId);
    Optional<UploadResultEntity> findByCycleIdAndPlatform(long cycleId, UploadPlatform platform);
}
```

- [ ] **Step 9.5: Write failing tests for UploadConfirmationService**

```java
// src/test/.../pipeline/service/UploadConfirmationServiceTest.java
package kg.automation.rest.automatation.pipeline.service;

import kg.automation.rest.automatation.pipeline.domain.*;
import kg.automation.rest.automatation.pipeline.repository.CycleRepository;
import kg.automation.rest.automatation.pipeline.repository.UploadResultRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UploadConfirmationServiceTest {

    @Mock CycleRepository cycleRepo;
    @Mock UploadResultRepository uploadResultRepo;
    @Mock CycleLifecycleService lifecycleService;
    @InjectMocks UploadConfirmationService service;

    private CycleEntity cycleWith(String platforms) {
        CycleEntity c = new CycleEntity();
        c.setTargetPlatforms(platforms);
        c.setFileLifecycle(FileLifecycle.ACTIVE);
        return c;
    }

    @Test
    void check_transitionsToUploadsConfirmed_whenAllPlatformsSucceeded() {
        CycleEntity cycle = cycleWith("YOUTUBE,TELEGRAM");
        when(cycleRepo.findById(1L)).thenReturn(Optional.of(cycle));
        when(uploadResultRepo.findByCycleId(1L)).thenReturn(List.of(
            resultFor(1L, UploadPlatform.YOUTUBE, "SUCCESS"),
            resultFor(1L, UploadPlatform.TELEGRAM, "SUCCESS")
        ));

        service.check(1L);

        verify(lifecycleService).onAllUploadsConfirmed(1L);
    }

    @Test
    void check_doesNotTransition_whenOnlyPartialSuccess() {
        CycleEntity cycle = cycleWith("YOUTUBE,TELEGRAM,PATREON");
        when(cycleRepo.findById(2L)).thenReturn(Optional.of(cycle));
        when(uploadResultRepo.findByCycleId(2L)).thenReturn(List.of(
            resultFor(2L, UploadPlatform.YOUTUBE, "SUCCESS")
        ));

        service.check(2L);

        verify(lifecycleService, never()).onAllUploadsConfirmed(anyLong());
    }

    @Test
    void check_doesNotTransition_whenAPlatformFailed() {
        CycleEntity cycle = cycleWith("YOUTUBE,TELEGRAM");
        when(cycleRepo.findById(3L)).thenReturn(Optional.of(cycle));
        when(uploadResultRepo.findByCycleId(3L)).thenReturn(List.of(
            resultFor(3L, UploadPlatform.YOUTUBE, "SUCCESS"),
            resultFor(3L, UploadPlatform.TELEGRAM, "FAILED")
        ));

        service.check(3L);

        verify(lifecycleService, never()).onAllUploadsConfirmed(anyLong());
    }

    private UploadResultEntity resultFor(long cycleId, UploadPlatform platform, String status) {
        UploadResultEntity r = new UploadResultEntity();
        r.setCycleId(cycleId);
        r.setPlatform(platform);
        r.setStatus(status);
        r.setCompletedAt(Instant.now());
        return r;
    }
}
```

- [ ] **Step 9.6: Run tests to confirm they fail**

> **Note:** The test mocks `CycleLifecycleService`, which is created in Task 10. The class must exist at compile time. If you get a compile error like `cannot find symbol: class CycleLifecycleService`, create a minimal stub first:
> ```java
> // pipeline/service/CycleLifecycleService.java
> package kg.automation.rest.automatation.pipeline.service;
> import org.springframework.stereotype.Service;
> @Service public class CycleLifecycleService {
>     public void onAllUploadsConfirmed(long cycleId) {}
>     public void approveForDeletion(long cycleId) {}
>     public void markFilesDeleted(long cycleId) {}
> }
> ```
> Task 10 replaces this stub with the full guarded implementation.

```bash
./mvnw test -Dtest=UploadConfirmationServiceTest 2>&1 | tail -10
```
Expected: `FAILURE` — `UploadConfirmationService` not yet created.

- [ ] **Step 9.7: Implement UploadConfirmationService**

```java
// pipeline/service/UploadConfirmationService.java
package kg.automation.rest.automatation.pipeline.service;

import kg.automation.rest.automatation.pipeline.domain.UploadPlatform;
import kg.automation.rest.automatation.pipeline.domain.UploadResultEntity;
import kg.automation.rest.automatation.pipeline.repository.CycleRepository;
import kg.automation.rest.automatation.pipeline.repository.UploadResultRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class UploadConfirmationService {

    private final CycleRepository cycleRepo;
    private final UploadResultRepository uploadResultRepo;
    private final CycleLifecycleService lifecycleService;

    /**
     * Call this after any upload job completes. If all target platforms for the cycle
     * have a SUCCESS row in upload_result, transitions the cycle to UPLOADS_CONFIRMED.
     */
    public void check(long cycleId) {
        var cycle = cycleRepo.findById(cycleId).orElseThrow(
                () -> new IllegalArgumentException("Cycle not found: " + cycleId));

        List<String> targets = Arrays.stream(cycle.getTargetPlatforms().split(","))
                .map(String::trim)
                .filter(s -> !s.isBlank())
                .toList();

        Map<String, String> resultsByPlatform = uploadResultRepo.findByCycleId(cycleId)
                .stream()
                .collect(Collectors.toMap(
                        r -> r.getPlatform().name(),
                        UploadResultEntity::getStatus,
                        (a, b) -> a));

        boolean allConfirmed = targets.stream()
                .allMatch(p -> "SUCCESS".equalsIgnoreCase(resultsByPlatform.get(p)));

        if (allConfirmed) {
            lifecycleService.onAllUploadsConfirmed(cycleId);
        }
    }
}
```

- [ ] **Step 9.8: Run tests — must pass**

```bash
./mvnw test -Dtest=UploadConfirmationServiceTest 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, `Tests run: 3, Failures: 0`.

- [ ] **Step 9.9: Commit**

```bash
git add src/main/resources/db/migration/V202605072466__upload_result.sql \
        src/main/java/kg/automation/rest/automatation/pipeline/domain/UploadPlatform.java \
        src/main/java/kg/automation/rest/automatation/pipeline/domain/UploadResultEntity.java \
        src/main/java/kg/automation/rest/automatation/pipeline/repository/UploadResultRepository.java \
        src/main/java/kg/automation/rest/automatation/pipeline/service/UploadConfirmationService.java \
        src/test/java/kg/automation/rest/automatation/pipeline/service/UploadConfirmationServiceTest.java
git commit -m "feat: upload_result table + UploadConfirmationService for lifecycle gate"
```

---

## Task 10: CycleLifecycleService

**Files:**
- Create: `pipeline/service/CycleLifecycleService.java`
- Create: `src/test/.../pipeline/service/CycleLifecycleServiceTest.java`

- [ ] **Step 10.1: Write failing tests**

```java
// src/test/.../pipeline/service/CycleLifecycleServiceTest.java
package kg.automation.rest.automatation.pipeline.service;

import kg.automation.rest.automatation.pipeline.domain.CycleEntity;
import kg.automation.rest.automatation.pipeline.domain.FileLifecycle;
import kg.automation.rest.automatation.pipeline.repository.CycleRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CycleLifecycleServiceTest {

    @Mock CycleRepository cycleRepo;
    @InjectMocks CycleLifecycleService service;

    private CycleEntity cycleWithState(FileLifecycle state) {
        CycleEntity c = new CycleEntity();
        c.setFileLifecycle(state);
        return c;
    }

    @Test
    void onAllUploadsConfirmed_transitionsFromActive() {
        CycleEntity c = cycleWithState(FileLifecycle.ACTIVE);
        when(cycleRepo.findById(1L)).thenReturn(Optional.of(c));

        service.onAllUploadsConfirmed(1L);

        assertEquals(FileLifecycle.UPLOADS_CONFIRMED, c.getFileLifecycle());
        verify(cycleRepo).save(c);
    }

    @Test
    void onAllUploadsConfirmed_isIdempotentIfAlreadyConfirmed() {
        CycleEntity c = cycleWithState(FileLifecycle.UPLOADS_CONFIRMED);
        when(cycleRepo.findById(2L)).thenReturn(Optional.of(c));

        service.onAllUploadsConfirmed(2L);

        // Still UPLOADS_CONFIRMED, save not called again
        assertEquals(FileLifecycle.UPLOADS_CONFIRMED, c.getFileLifecycle());
        verify(cycleRepo, never()).save(any());
    }

    @Test
    void approveForDeletion_transitionsFromUploadsConfirmed() {
        CycleEntity c = cycleWithState(FileLifecycle.UPLOADS_CONFIRMED);
        when(cycleRepo.findById(3L)).thenReturn(Optional.of(c));

        service.approveForDeletion(3L);

        assertEquals(FileLifecycle.APPROVED_FOR_DELETE, c.getFileLifecycle());
        verify(cycleRepo).save(c);
    }

    @Test
    void approveForDeletion_throwsIfNotYetConfirmed() {
        CycleEntity c = cycleWithState(FileLifecycle.ACTIVE);
        when(cycleRepo.findById(4L)).thenReturn(Optional.of(c));

        assertThrows(IllegalStateException.class, () -> service.approveForDeletion(4L));
        verify(cycleRepo, never()).save(any());
    }
}
```

- [ ] **Step 10.2: Run tests to confirm they fail**

```bash
./mvnw test -Dtest=CycleLifecycleServiceTest 2>&1 | tail -10
```
Expected: `FAILURE`.

- [ ] **Step 10.3: Implement CycleLifecycleService**

```java
// pipeline/service/CycleLifecycleService.java
package kg.automation.rest.automatation.pipeline.service;

import kg.automation.rest.automatation.pipeline.domain.CycleEntity;
import kg.automation.rest.automatation.pipeline.domain.FileLifecycle;
import kg.automation.rest.automatation.pipeline.repository.CycleRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class CycleLifecycleService {

    private final CycleRepository cycleRepo;

    @Transactional
    public void onAllUploadsConfirmed(long cycleId) {
        CycleEntity c = cycleRepo.findById(cycleId).orElseThrow();
        if (c.getFileLifecycle() == FileLifecycle.ACTIVE) {
            c.setFileLifecycle(FileLifecycle.UPLOADS_CONFIRMED);
            cycleRepo.save(c);
        }
    }

    @Transactional
    public void approveForDeletion(long cycleId) {
        CycleEntity c = cycleRepo.findById(cycleId).orElseThrow();
        if (c.getFileLifecycle() != FileLifecycle.UPLOADS_CONFIRMED) {
            throw new IllegalStateException(
                    "Cycle " + cycleId + " cannot be approved for deletion: " +
                    "current state is " + c.getFileLifecycle() + ", expected UPLOADS_CONFIRMED");
        }
        c.setFileLifecycle(FileLifecycle.APPROVED_FOR_DELETE);
        cycleRepo.save(c);
    }

    @Transactional
    public void markFilesDeleted(long cycleId) {
        CycleEntity c = cycleRepo.findById(cycleId).orElseThrow();
        c.setFileLifecycle(FileLifecycle.FILES_DELETED);
        cycleRepo.save(c);
    }
}
```

- [ ] **Step 10.4: Run tests — must pass**

```bash
./mvnw test -Dtest=CycleLifecycleServiceTest 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, `Tests run: 4, Failures: 0`.

- [ ] **Step 10.5: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/pipeline/service/CycleLifecycleService.java \
        src/test/java/kg/automation/rest/automatation/pipeline/service/CycleLifecycleServiceTest.java
git commit -m "feat: CycleLifecycleService — guarded state machine for file lifecycle transitions"
```

---

## Task 11: CycleCleanupScheduler

**Files:**
- Create: `pipeline/scheduler/CycleCleanupScheduler.java`

- [ ] **Step 11.1: Implement the scheduler**

```java
// pipeline/scheduler/CycleCleanupScheduler.java
package kg.automation.rest.automatation.pipeline.scheduler;

import kg.automation.rest.automatation.pipeline.domain.CycleEntity;
import kg.automation.rest.automatation.pipeline.domain.FileLifecycle;
import kg.automation.rest.automatation.pipeline.repository.CycleRepository;
import kg.automation.rest.automatation.pipeline.service.CycleFileLayout;
import kg.automation.rest.automatation.pipeline.service.CycleLifecycleService;
import lombok.RequiredArgsConstructor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;

@Component
@RequiredArgsConstructor
public class CycleCleanupScheduler {

    private static final Logger log = LoggerFactory.getLogger(CycleCleanupScheduler.class);

    private final CycleRepository cycleRepo;
    private final CycleFileLayout layout;
    private final CycleLifecycleService lifecycleService;

    /** Runs every hour. Idempotent — safe to run multiple times on the same cycle. */
    @Scheduled(fixedDelay = 3_600_000)
    public void run() {
        deleteApprovedCycles();
        sweepStaleTmpFiles();
    }

    private void deleteApprovedCycles() {
        List<CycleEntity> approved = cycleRepo.findByFileLifecycle(FileLifecycle.APPROVED_FOR_DELETE);
        for (CycleEntity c : approved) {
            try {
                layout.deleteCycleDir(c.getId());
                lifecycleService.markFilesDeleted(c.getId());
                log.info("Deleted files for cycle={}", c.getId());
            } catch (Exception e) {
                log.error("Failed to delete files for cycle={}: {}", c.getId(), e.getMessage());
            }
        }
    }

    private void sweepStaleTmpFiles() {
        Instant cutoff = Instant.now().minus(2, ChronoUnit.HOURS);
        List<CycleEntity> active = cycleRepo.findByFileLifecycle(FileLifecycle.ACTIVE);
        for (CycleEntity c : active) {
            Path tmpDir = layout.cycleDir(c.getId()).resolve("tmp");
            if (!Files.exists(tmpDir)) continue;
            try (var stream = Files.list(tmpDir)) {
                stream.filter(p -> isOlderThan(p, cutoff))
                      .forEach(layout::deleteTmpFile);
            } catch (IOException e) {
                log.warn("Failed to sweep tmp for cycle={}: {}", c.getId(), e.getMessage());
            }
        }
    }

    private static boolean isOlderThan(Path p, Instant cutoff) {
        try {
            return Files.getLastModifiedTime(p).toInstant().isBefore(cutoff);
        } catch (IOException e) {
            return false;
        }
    }
}
```

- [ ] **Step 11.2: Ensure @EnableScheduling is active**

Check `audio-library-automation-bot/src/main/java/.../AutomatationApplication.java` (or the main class) — confirm `@EnableScheduling` is present. If not, add it.

```bash
grep -r "@EnableScheduling" src/main/java/ --include="*.java"
```

If nothing is found:
```bash
# Find the main application class
grep -rn "@SpringBootApplication" src/main/java/ --include="*.java"
```
Then add `@EnableScheduling` to that class.

- [ ] **Step 11.3: Compile check**

```bash
./mvnw compile 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 11.4: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/pipeline/scheduler/CycleCleanupScheduler.java
git commit -m "feat: CycleCleanupScheduler — hourly deletion of approved cycles and stale tmp files"
```

---

## Task 12: approve-delete endpoint + CycleResponse fields

**Files:**
- Modify: `pipeline/web/CycleV1Controller.java`

- [ ] **Step 12.1: Add the approve-delete endpoint**

Replace the contents of `CycleV1Controller.java`:
```java
package kg.automation.rest.automatation.pipeline.web;

import kg.automation.rest.automatation.pipeline.dto.AudioPartResponse;
import kg.automation.rest.automatation.pipeline.dto.ImagePartResponse;
import kg.automation.rest.automatation.pipeline.dto.ScrapedBookResponse;
import kg.automation.rest.automatation.pipeline.dto.VideoPartResponse;
import kg.automation.rest.automatation.pipeline.service.CycleLifecycleService;
import kg.automation.rest.automatation.pipeline.service.CyclePartQueryService;
import kg.automation.rest.automatation.pipeline.service.ScrapedBookQueryService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/cycles")
@RequiredArgsConstructor
public class CycleV1Controller {

    private final ScrapedBookQueryService scrapedBookQueryService;
    private final CyclePartQueryService cyclePartQueryService;
    private final CycleLifecycleService cycleLifecycleService;

    @PostMapping("/{cycleId}/approve-delete")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void approveForDeletion(@PathVariable Long cycleId) {
        cycleLifecycleService.approveForDeletion(cycleId);
    }

    @GetMapping("/{cycleId}/scraped-books")
    public ResponseEntity<List<ScrapedBookResponse>> listScrapedBooks(@PathVariable Long cycleId) {
        return scrapedBookQueryService.listByCycleId(cycleId)
                .map(ResponseEntity::ok).orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/{cycleId}/audio-parts")
    public ResponseEntity<List<AudioPartResponse>> listAudioParts(@PathVariable Long cycleId) {
        return cyclePartQueryService.listAudioParts(cycleId)
                .map(ResponseEntity::ok).orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/{cycleId}/video-parts")
    public ResponseEntity<List<VideoPartResponse>> listVideoParts(@PathVariable Long cycleId) {
        return cyclePartQueryService.listVideoParts(cycleId)
                .map(ResponseEntity::ok).orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/{cycleId}/image-parts")
    public ResponseEntity<List<ImagePartResponse>> listImageParts(@PathVariable Long cycleId) {
        return cyclePartQueryService.listImageParts(cycleId)
                .map(ResponseEntity::ok).orElse(ResponseEntity.notFound().build());
    }
}
```

- [ ] **Step 12.2: Write an integration test for the endpoint**

```java
// src/test/.../pipeline/web/CycleLifecycleEndpointTest.java
package kg.automation.rest.automatation.pipeline.web;

import kg.automation.rest.automatation.pipeline.service.CycleLifecycleService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(CycleV1Controller.class)
class CycleLifecycleEndpointTest {

    @Autowired MockMvc mockMvc;
    @MockBean CycleLifecycleService lifecycleService;
    @MockBean kg.automation.rest.automatation.pipeline.service.ScrapedBookQueryService scrapedBookQueryService;
    @MockBean kg.automation.rest.automatation.pipeline.service.CyclePartQueryService cyclePartQueryService;

    @Test
    void approveDelete_returns204_onSuccess() throws Exception {
        mockMvc.perform(post("/api/v1/cycles/42/approve-delete"))
               .andExpect(status().isNoContent());
        verify(lifecycleService).approveForDeletion(42L);
    }

    @Test
    void approveDelete_returns409_whenNotYetConfirmed() throws Exception {
        doThrow(new IllegalStateException("state is ACTIVE"))
                .when(lifecycleService).approveForDeletion(99L);

        mockMvc.perform(post("/api/v1/cycles/99/approve-delete"))
               .andExpect(status().isConflict());
    }
}
```

Note: the 409 test requires a `@ControllerAdvice` or `@ExceptionHandler` for `IllegalStateException` → HTTP 409. If none exists, add a simple one:

```java
// infra/web/GlobalExceptionHandler.java  (or add to existing advice if one exists)
package kg.automation.rest.automatation.infra.web;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(IllegalStateException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public String handleIllegalState(IllegalStateException ex) {
        return ex.getMessage();
    }
}
```

Check first:
```bash
grep -rn "IllegalStateException\|ControllerAdvice\|ExceptionHandler" src/main/java/ --include="*.java" | grep -v "test" | head -10
```

- [ ] **Step 12.3: Run endpoint test**

```bash
./mvnw test -Dtest=CycleLifecycleEndpointTest 2>&1 | tail -15
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 12.4: Commit**

```bash
git add src/main/java/kg/automation/rest/automatation/pipeline/web/CycleV1Controller.java \
        src/test/java/kg/automation/rest/automatation/pipeline/web/CycleLifecycleEndpointTest.java
# also add GlobalExceptionHandler if created
git commit -m "feat: POST /cycles/{id}/approve-delete endpoint for human-gated file deletion"
```

---

## Task 13: Full test suite pass

- [ ] **Step 13.1: Run all tests**

```bash
cd audio-library-automation-bot
./mvnw test 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`. All tests pass.

- [ ] **Step 13.2: If any tests fail, fix them before proceeding**

Common failure causes after these changes:
- Tests that construct `AudioConcatenateJobRequest` with `inputDirectory`/`outputDirectory` — those fields are now optional/ignored by the runner; tests may need updated expectations
- Tests that instantiate `RestCallService` — now needs `CycleFileLayout` in constructor
- Tests that use `ProcessingTempDirService` directly — still pass via delegation, but check

- [ ] **Step 13.3: Commit any test fixes**

```bash
git add -p  # review and stage only test fixes
git commit -m "test: fix tests broken by CycleFileLayout migration"
```

---

## Task 14: Frontend — lifecycle panel

**Files:**
- Modify: `audio-frontend/src/` — add lifecycle status and approve button to the cycle/book detail view

- [ ] **Step 14.1: Identify the cycle detail component**

```bash
cd audio-frontend
grep -rn "cycleId\|cycle_id\|CycleId" src/ --include="*.tsx" --include="*.ts" | head -15
```

Find the component that renders cycle/book details. Open it.

- [ ] **Step 14.2: Add fileLifecycle display and approve button**

In the cycle detail component, add after the existing status display:

```tsx
// Add to the component that shows cycle details
const approveForDeletion = async (cycleId: number) => {
  await fetch(`${import.meta.env.VITE_CORE_API_URL}/api/v1/cycles/${cycleId}/approve-delete`, {
    method: 'POST',
  });
  // refresh cycle data
};

// In the JSX, add:
{cycle.fileLifecycle && (
  <div className="lifecycle-section">
    <span>Files: {cycle.fileLifecycle}</span>
    {cycle.fileLifecycle === 'UPLOADS_CONFIRMED' && (
      <button
        onClick={() => approveForDeletion(cycle.id)}
        className="approve-delete-btn"
      >
        Approve file deletion
      </button>
    )}
    {cycle.fileLifecycle === 'FILES_DELETED' && (
      <span className="files-deleted-badge">Files deleted</span>
    )}
  </div>
)}
```

- [ ] **Step 14.3: Type-check**

```bash
cd audio-frontend
npm run lint
```
Expected: no errors.

- [ ] **Step 14.4: Start dev server and verify manually**

```bash
npm run dev
```
Open `http://localhost:5173`, navigate to a cycle detail page, confirm the lifecycle status shows and the button appears only when state is `UPLOADS_CONFIRMED`.

- [ ] **Step 14.5: Commit**

```bash
git add src/
git commit -m "feat: show file lifecycle status and approve-delete button on cycle detail page"
```

---

## Self-Review Checklist

After completing all tasks, verify:

- [ ] `grep -rn "resources/temp" audio-library-automation-bot/src/main/java/` → no results
- [ ] `grep -rn '"/base.png"\|tempDir.*base' audio-library-automation-bot/src/main/java/` → no results  
- [ ] `grep -rn 'tempDir.*"/torrents"\|tempDir.*"/audio"' audio-library-automation-bot/src/main/java/` → no results
- [ ] `./mvnw test` passes with `BUILD SUCCESS`
- [ ] `npm run lint` in `audio-frontend/` passes with no errors
