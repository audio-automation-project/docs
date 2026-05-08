# Firebase Full Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace PostgreSQL + local filesystem with Firestore-only persistence; audio/video files stay ephemeral (temp-only, deleted after distribution).

> **Superseded vs current repo:** The codebase uses **PostgreSQL + Flyway** as system of record (including **`library_audiobooks`**, **`job_logs`**). **`DistributedToDto`** is **`youtubeVideoId`** + **`telegramFileId`** only (see Flyway **`V202605052210__library_audiobooks.sql`**). Keep this file as historical migration planning; align any revived work with the live schema and DTOs.

**Architecture:** Spring Boot remains as a stateless processing engine running FFmpeg/Playwright. All metadata, job status, and chunked images live in Firestore. Audio files are downloaded to a configurable temp dir, processed, distributed, then deleted. Any machine with a Firebase service account JSON can run the system.

**Tech Stack:** Java 17, Spring Boot 3.4, firebase-admin 9.4.3 (already in pom), React 19, TypeScript, firebase npm SDK, Python 3 (migration script)

---

## File Map

### New files
| File | Responsibility |
|------|---------------|
| `firebase/service/ProcessingTempDirService.java` | Creates/deletes job temp dirs under `processing.temp.dir` |
| `firebase/service/FirestoreJobLogService.java` | Writes STARTED/COMPLETED/FAILED to `/job_logs/{jobId}` |
| `firebase/service/FirestoreAudiobookService.java` | CRUD on `/audiobooks` — replaces `AudiobookLibraryService` |
| `firebase/service/FirestoreImageService.java` | Compress → JPEG → chunk → write `/image_store` + `/image_chunks` |
| `firebase/service/FirestoreAiLogService.java` | Replaces `AiTraceService` — logs to `/ai_integration_log` + `/ai_traces` |
| `library/dto/DistributedToDto.java` | Embedded DTO: `youtubeVideoId`, `telegramFileId` (historical plan also mentioned Boosty — dropped in current Postgres catalog) |
| `audio-frontend/src/firebase.ts` | Firebase app + Firestore client init |
| `scripts/migrate-to-firestore.py` | One-shot migration: PostgreSQL JSON dumps → Firestore |

### Modified files
| File | Change |
|------|--------|
| `audio-library-automation-bot/pom.xml` | Remove JPA/PG/Flyway/H2 |
| `src/main/resources/application.properties` | Remove DB config; add `processing.temp.dir`, `openai.api.key`, `search.api.token` |
| `firebase/FirebaseConfig.java` | Remove `@ConditionalOnProperty` — Firebase always on |
| `library/dto/AudiobookDto.java` | Replace `audioFilePath`/`coverImagePath` with `coverImageIds`, `distributedTo`, `processingStatus` |
| `library/web/AudiobookV1Controller.java` | Inject `FirestoreAudiobookService` instead of `AudiobookLibraryService` |
| `audio/service/AudioConcatenatorService.java` | Replace `AudioDescriptionPersistenceService` with `FirestoreJobLogService` |
| `audio/service/AudioCompressorService.java` | Inject `FirestoreJobLogService`; log STARTED/COMPLETED |
| `image/service/BookCoverService.java` | After generating cover to temp dir, store in Firestore via `FirestoreImageService` |
| `gateway/service/RestCallService.java` | Replace hardcoded API key string with `@Value("${openai.api.key}")` |
| `audio-frontend/src/api/client.ts` | Add Firestore `onSnapshot` listener on `/job_logs` |
| `audio-frontend/src/App.tsx` | Render live job log panel from Firestore |
| `audio-frontend/package.json` | Add `firebase` npm dependency |

### Deleted files
| File | Reason |
|------|--------|
| `library/domain/AudiobookEntity.java` | Replaced by Firestore |
| `library/domain/AudioDescriptionEntity.java` | Replaced by `/job_logs` |
| `library/domain/AiIntegrationLogEntity.java` | Replaced by `/ai_integration_log` |
| `library/repository/AudiobookRepository.java` | JPA gone |
| `library/repository/AudioDescriptionRepository.java` | JPA gone |
| `library/repository/AiIntegrationLogRepository.java` | JPA gone |
| `library/service/AudiobookLibraryService.java` | Replaced by `FirestoreAudiobookService` |
| `library/service/AudioDescriptionPersistenceService.java` | Replaced by `FirestoreJobLogService` |
| `library/service/AiTraceService.java` | Replaced by `FirestoreAiLogService` |
| `library/service/JsonAudiobookImportService.java` | One-shot migration, no longer needed |
| `library/service/JsonMediaGroupImportService.java` | One-shot migration, no longer needed |
| `file/service/ProperityConfig.java` | Replaced by `processing.temp.dir` property + `ProcessingTempDirService` |
| Tests: `AudiobookRepositoryTest`, `JsonAudiobookImportServiceTest`, `AiTraceServiceTest`, `MediaGroupRepositoryTest`, `AudioDescriptionPersistenceServiceTest` | Classes deleted |

All Java paths are relative to `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/`.
All test paths are relative to `audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/`.

---

## Phase 1 — Remove Old Infrastructure

### Task 1: Strip JPA/PostgreSQL/Flyway from pom.xml

**Files:**
- Modify: `audio-library-automation-bot/pom.xml`

- [ ] **Step 1: Remove the four dependencies**

Open `pom.xml` and delete these four `<dependency>` blocks:

```xml
<!-- DELETE THIS -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- DELETE THIS -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- DELETE THIS -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>

<!-- DELETE THIS -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>

<!-- DELETE THIS -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

- [ ] **Step 2: Verify firebase-admin is present**

Confirm this block exists in `pom.xml` (do not add, just verify):
```xml
<dependency>
    <groupId>com.google.firebase</groupId>
    <artifactId>firebase-admin</artifactId>
    <version>9.4.3</version>
</dependency>
```

- [ ] **Step 3: Attempt compile to reveal all broken imports**

```bash
cd audio-library-automation-bot
mvn compile -q 2>&1 | grep "error:" | head -40
```

Expected: Many errors about missing JPA/entity classes — that's correct. These will be fixed as we delete the old code in Phase 1.

---

### Task 2: Update application.properties

**Files:**
- Modify: `audio-library-automation-bot/src/main/resources/application.properties`

- [ ] **Step 1: Replace the file content**

Replace the entire file with:

```properties
spring.application.name=Automatation

# Local overrides (never commit secrets)
spring.config.import=optional:file:./application-local.properties,optional:file:./config/application-local.properties

server.port=${SERVER_PORT:8088}

# --- Firebase (always enabled) ---
firebase.enabled=true
firebase.project-id=${FIREBASE_PROJECT_ID:}
firebase.firestore.sync=true
firebase.firestore.collection=audiobooks

# --- Processing temp dir (any writable path; defaults to OS temp) ---
processing.temp.dir=${PROCESSING_TEMP_DIR:${java.io.tmpdir}/audio-pipeline}

# --- OpenAI ---
openai.api.key=${OPENAI_API_KEY:}

# --- Local LLM / search proxy ---
search.api.token=${SEARCH_API_TOKEN:}
search.api.url=${SEARCH_API_URL:http://localhost:8080}

# --- qBittorrent Web UI ---
torrent.api.url=${TORRENT_API_URL:http://localhost:8085}
torrent.api.username=${TORRENT_API_USERNAME:admin}
torrent.api.password=${TORRENT_API_PASSWORD:admin123}

# --- Telegram Bot API ---
telegram.bot.username=${TELEGRAM_BOT_USERNAME:AudioLibraryDevBot}
telegram.bot.token=${TELEGRAM_BOT_TOKEN:000000000:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA}

# --- Actuator ---
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=never
management.endpoints.web.base-path=/actuator
```

- [ ] **Step 2: Delete the H2 and dev profile properties files**

```bash
rm -f audio-library-automation-bot/src/main/resources/application-h2.properties
rm -f audio-library-automation-bot/src/main/resources/application-dev.properties
```

---

### Task 3: Collapse FirebaseConfig — always enabled

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/FirebaseConfig.java`

- [ ] **Step 1: Write the failing test**

Create `src/test/java/kg/automation/rest/automatation/firebase/FirebaseConfigTest.java`:

```java
package kg.automation.rest.automatation.firebase;

import com.google.cloud.firestore.Firestore;
import com.google.firebase.FirebaseApp;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import org.springframework.beans.factory.annotation.Autowired;

@SpringBootTest(properties = {
    "GOOGLE_APPLICATION_CREDENTIALS=src/test/resources/test-service-account.json",
    "firebase.project-id=test-project"
})
class FirebaseConfigTest {

    @Autowired(required = false)
    Firestore firestore;

    @Test
    void firestoreBeanExistsWithoutFeatureFlag() {
        // After collapsing @ConditionalOnProperty, Firestore bean is always created
        assertNotNull(firestore, "Firestore bean must be registered unconditionally");
    }
}
```

Note: This test requires a `test-service-account.json`. Create a minimal stub:

```bash
mkdir -p audio-library-automation-bot/src/test/resources
cat > audio-library-automation-bot/src/test/resources/test-service-account.json << 'EOF'
{
  "type": "service_account",
  "project_id": "test-project",
  "private_key_id": "key1",
  "private_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEA2a2rwplBQLF29amygykEMmYz0+Kcj3bKBp29rNcNMnFQ\n-----END RSA PRIVATE KEY-----\n",
  "client_email": "test@test-project.iam.gserviceaccount.com",
  "client_id": "123",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token"
}
EOF
```

For unit tests that don't need real Firebase, use `@MockitoBean` on `Firestore` — see Task 6.

- [ ] **Step 2: Replace FirebaseConfig.java**

```java
package kg.automation.rest.automatation.firebase;

import com.google.auth.oauth2.GoogleCredentials;
import com.google.cloud.firestore.Firestore;
import com.google.firebase.FirebaseApp;
import com.google.firebase.FirebaseOptions;
import com.google.firebase.cloud.FirestoreClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.IOException;

@Configuration
public class FirebaseConfig {

    @Bean
    public FirebaseApp firebaseApp(FirebaseProperties properties) throws IOException {
        if (!FirebaseApp.getApps().isEmpty()) {
            return FirebaseApp.getInstance();
        }
        FirebaseOptions.Builder builder = FirebaseOptions.builder()
                .setCredentials(GoogleCredentials.getApplicationDefault());
        if (properties.getProjectId() != null && !properties.getProjectId().isBlank()) {
            builder.setProjectId(properties.getProjectId().trim());
        }
        return FirebaseApp.initializeApp(builder.build());
    }

    @Bean
    public Firestore firestore(FirebaseApp firebaseApp) {
        return FirestoreClient.getFirestore(firebaseApp);
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/FirebaseConfig.java \
        audio-library-automation-bot/src/main/resources/application.properties \
        audio-library-automation-bot/pom.xml
git commit -m "feat: remove PostgreSQL/JPA/Flyway deps; Firebase always enabled"
```

---

## Phase 2 — New Firestore Services

### Task 4: ProcessingTempDirService

**Files:**
- Create: `firebase/service/ProcessingTempDirService.java`
- Create: `src/test/java/.../firebase/service/ProcessingTempDirServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
package kg.automation.rest.automatation.firebase.service;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import static org.junit.jupiter.api.Assertions.*;

class ProcessingTempDirServiceTest {

    @TempDir
    Path tempDir;

    @Test
    void createJobDir_createsSubdirectories() throws IOException {
        ProcessingTempDirService service = new ProcessingTempDirService(tempDir.toString());
        Path jobDir = service.createJobDir("job-001");

        assertTrue(Files.isDirectory(jobDir.resolve("source")));
        assertTrue(Files.isDirectory(jobDir.resolve("compressed")));
        assertTrue(Files.isDirectory(jobDir.resolve("covers")));
        assertTrue(Files.isDirectory(jobDir.resolve("video")));
    }

    @Test
    void cleanupJobDir_deletesEntireTree() throws IOException {
        ProcessingTempDirService service = new ProcessingTempDirService(tempDir.toString());
        Path jobDir = service.createJobDir("job-002");
        Files.writeString(jobDir.resolve("source/test.mp3"), "data");

        service.cleanupJobDir("job-002");

        assertFalse(Files.exists(jobDir));
    }
}
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd audio-library-automation-bot
mvn test -Dtest=ProcessingTempDirServiceTest -q 2>&1 | tail -5
```

Expected: compilation error — class does not exist.

- [ ] **Step 3: Implement**

```java
package kg.automation.rest.automatation.firebase.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Comparator;
import java.util.stream.Stream;

@Service
public class ProcessingTempDirService {

    private final String baseDir;

    public ProcessingTempDirService(@Value("${processing.temp.dir:${java.io.tmpdir}/audio-pipeline}") String baseDir) {
        this.baseDir = baseDir;
    }

    public Path createJobDir(String jobId) throws IOException {
        Path jobDir = Path.of(baseDir, jobId);
        Files.createDirectories(jobDir.resolve("source"));
        Files.createDirectories(jobDir.resolve("compressed"));
        Files.createDirectories(jobDir.resolve("covers"));
        Files.createDirectories(jobDir.resolve("video"));
        return jobDir;
    }

    public Path jobDir(String jobId) {
        return Path.of(baseDir, jobId);
    }

    public void cleanupJobDir(String jobId) {
        Path jobDir = Path.of(baseDir, jobId);
        if (!Files.exists(jobDir)) return;
        try (Stream<Path> walk = Files.walk(jobDir)) {
            walk.sorted(Comparator.reverseOrder())
                .forEach(p -> p.toFile().delete());
        } catch (IOException e) {
            System.err.println("Warning: failed to cleanup temp dir " + jobDir + ": " + e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run — confirm PASS**

```bash
mvn test -Dtest=ProcessingTempDirServiceTest -q 2>&1 | tail -5
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/service/ProcessingTempDirService.java \
        audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/firebase/service/ProcessingTempDirServiceTest.java
git commit -m "feat: add ProcessingTempDirService for ephemeral job temp dirs"
```

---

### Task 5: FirestoreJobLogService

**Files:**
- Create: `firebase/service/FirestoreJobLogService.java`
- Create: `src/test/java/.../firebase/service/FirestoreJobLogServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
package kg.automation.rest.automatation.firebase.service;

import com.google.api.core.ApiFuture;
import com.google.cloud.firestore.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class FirestoreJobLogServiceTest {

    Firestore firestore;
    CollectionReference collection;
    DocumentReference docRef;
    ApiFuture<WriteResult> future;
    FirestoreJobLogService service;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        firestore = mock(Firestore.class);
        collection = mock(CollectionReference.class);
        docRef = mock(DocumentReference.class);
        future = mock(ApiFuture.class);
        when(firestore.collection("job_logs")).thenReturn(collection);
        when(collection.document(anyString())).thenReturn(docRef);
        when(docRef.set(anyMap())).thenReturn(future);
        when(docRef.update(anyMap())).thenReturn(future);
        service = new FirestoreJobLogService(firestore);
    }

    @Test
    void startJob_writesStartedStatus() {
        service.startJob("job-1", "COMPRESS", "book-123");

        @SuppressWarnings("unchecked")
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
        verify(docRef).set(captor.capture());
        assertEquals("STARTED", captor.getValue().get("status"));
        assertEquals("COMPRESS", captor.getValue().get("type"));
        assertEquals("book-123", captor.getValue().get("audiobookId"));
    }

    @Test
    void completeJob_writesCompletedStatus() {
        service.completeJob("job-1", "done");

        @SuppressWarnings("unchecked")
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
        verify(docRef).update(captor.capture());
        assertEquals("COMPLETED", captor.getValue().get("status"));
    }

    @Test
    void failJob_writesFailedStatusWithDetail() {
        service.failJob("job-1", "failed", "NPE at line 42");

        @SuppressWarnings("unchecked")
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
        verify(docRef).update(captor.capture());
        assertEquals("FAILED", captor.getValue().get("status"));
        assertEquals("NPE at line 42", captor.getValue().get("errorDetail"));
    }
}
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
mvn test -Dtest=FirestoreJobLogServiceTest -q 2>&1 | tail -5
```

- [ ] **Step 3: Implement**

```java
package kg.automation.rest.automatation.firebase.service;

import com.google.cloud.firestore.FieldValue;
import com.google.cloud.firestore.Firestore;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

@Service
public class FirestoreJobLogService {

    private static final String COLLECTION = "job_logs";

    private final Firestore firestore;

    public FirestoreJobLogService(Firestore firestore) {
        this.firestore = firestore;
    }

    public void startJob(String jobId, String type, String audiobookId) {
        Map<String, Object> data = new HashMap<>();
        data.put("type", type);
        data.put("status", "STARTED");
        data.put("audiobookId", audiobookId != null ? audiobookId : "");
        data.put("message", "");
        data.put("errorDetail", null);
        data.put("startedAt", FieldValue.serverTimestamp());
        data.put("finishedAt", null);
        firestore.collection(COLLECTION).document(jobId).set(data);
    }

    public void completeJob(String jobId, String message) {
        Map<String, Object> data = new HashMap<>();
        data.put("status", "COMPLETED");
        data.put("message", message != null ? message : "");
        data.put("finishedAt", FieldValue.serverTimestamp());
        firestore.collection(COLLECTION).document(jobId).update(data);
    }

    public void failJob(String jobId, String message, String errorDetail) {
        Map<String, Object> data = new HashMap<>();
        data.put("status", "FAILED");
        data.put("message", message != null ? message : "");
        data.put("errorDetail", errorDetail);
        data.put("finishedAt", FieldValue.serverTimestamp());
        firestore.collection(COLLECTION).document(jobId).update(data);
    }
}
```

- [ ] **Step 4: Run — confirm PASS**

```bash
mvn test -Dtest=FirestoreJobLogServiceTest -q 2>&1 | tail -5
```

- [ ] **Step 5: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/service/FirestoreJobLogService.java \
        audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/firebase/service/FirestoreJobLogServiceTest.java
git commit -m "feat: add FirestoreJobLogService for pipeline job tracking"
```

---

### Task 6: AudiobookDto — update fields for Firestore model

**Files:**
- Create: `library/dto/DistributedToDto.java`
- Modify: `library/dto/AudiobookDto.java`

- [ ] **Step 1: Create DistributedToDto**

```java
package kg.automation.rest.automatation.library.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DistributedToDto {
    private String boostyUrl;
    private String telegramFileId;
    private String youtubeVideoId;
}
```

- [ ] **Step 2: Update AudiobookDto**

Find `AudiobookDto.java` at `library/dto/AudiobookDto.java`. Replace its content:

```java
package kg.automation.rest.automatation.library.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AudiobookDto {
    private String id;           // Firestore document ID (String, not Long)
    private String fileId;
    private String title;
    private String author;
    private String narrator;
    private String originalTitle;
    private String part;
    private String sourceLink;
    private Long duration;
    private String uploadDate;
    private String description;

    /** IDs of cover image documents in /image_store collection */
    private List<String> coverImageIds;

    /** Where this book has been distributed */
    private DistributedToDto distributedTo;

    /** IDLE | PROCESSING | DISTRIBUTED | FAILED */
    private String processingStatus;
}
```

Note: `id` changes from `Long` to `String` (Firestore document ID). Any existing callers that set `id` as Long must be updated.

- [ ] **Step 3: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/dto/
git commit -m "feat: update AudiobookDto for Firestore model — add coverImageIds, distributedTo, processingStatus"
```

---

### Task 7: FirestoreAudiobookService

**Files:**
- Create: `firebase/service/FirestoreAudiobookService.java`
- Create: `src/test/java/.../firebase/service/FirestoreAudiobookServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
package kg.automation.rest.automatation.firebase.service;

import com.google.api.core.ApiFutures;
import com.google.cloud.firestore.*;
import kg.automation.rest.automatation.library.dto.AudiobookDto;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class FirestoreAudiobookServiceTest {

    Firestore firestore;
    CollectionReference collection;
    DocumentReference docRef;
    QuerySnapshot querySnapshot;
    Query query;
    FirestoreAudiobookService service;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() throws Exception {
        firestore = mock(Firestore.class);
        collection = mock(CollectionReference.class);
        docRef = mock(DocumentReference.class);
        querySnapshot = mock(QuerySnapshot.class);
        query = mock(Query.class);
        when(firestore.collection("audiobooks")).thenReturn(collection);
        when(collection.document(anyString())).thenReturn(docRef);
        when(docRef.set(anyMap())).thenReturn(ApiFutures.immediateFuture(mock(WriteResult.class)));
        when(docRef.update(anyMap())).thenReturn(ApiFutures.immediateFuture(mock(WriteResult.class)));
        when(docRef.delete()).thenReturn(ApiFutures.immediateFuture(mock(WriteResult.class)));
        service = new FirestoreAudiobookService(firestore);
    }

    @Test
    void save_setsProcessingStatusIdle() throws Exception {
        AudiobookDto dto = AudiobookDto.builder().title("Test Book").build();

        AudiobookDto result = service.save(dto);

        assertNotNull(result.getId());
        assertEquals("IDLE", result.getProcessingStatus());
    }

    @Test
    void findById_returnsEmptyWhenNotFound() throws Exception {
        DocumentSnapshot snap = mock(DocumentSnapshot.class);
        when(snap.exists()).thenReturn(false);
        when(docRef.get()).thenReturn(ApiFutures.immediateFuture(snap));

        Optional<AudiobookDto> result = service.findById("nonexistent");

        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
mvn test -Dtest=FirestoreAudiobookServiceTest -q 2>&1 | tail -5
```

- [ ] **Step 3: Implement**

```java
package kg.automation.rest.automatation.firebase.service;

import com.google.cloud.firestore.*;
import kg.automation.rest.automatation.library.dto.AudiobookDto;
import kg.automation.rest.automatation.library.dto.DistributedToDto;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.concurrent.ExecutionException;
import java.util.stream.Collectors;

@Service
public class FirestoreAudiobookService {

    private static final String COLLECTION = "audiobooks";

    private final Firestore firestore;

    public FirestoreAudiobookService(Firestore firestore) {
        this.firestore = firestore;
    }

    public List<AudiobookDto> findAll() throws ExecutionException, InterruptedException {
        QuerySnapshot snap = firestore.collection(COLLECTION).get().get();
        return snap.getDocuments().stream()
                .map(this::fromDoc)
                .collect(Collectors.toList());
    }

    public Optional<AudiobookDto> findById(String id) throws ExecutionException, InterruptedException {
        DocumentSnapshot snap = firestore.collection(COLLECTION).document(id).get().get();
        return snap.exists() ? Optional.of(fromDoc(snap)) : Optional.empty();
    }

    public AudiobookDto save(AudiobookDto dto) throws ExecutionException, InterruptedException {
        String id = (dto.getId() != null && !dto.getId().isBlank()) ? dto.getId() : UUID.randomUUID().toString();
        Map<String, Object> data = toMap(dto);
        data.put("processingStatus", dto.getProcessingStatus() != null ? dto.getProcessingStatus() : "IDLE");
        data.put("createdAt", FieldValue.serverTimestamp());
        data.put("updatedAt", FieldValue.serverTimestamp());
        firestore.collection(COLLECTION).document(id).set(data).get();
        dto.setId(id);
        if (dto.getProcessingStatus() == null) dto.setProcessingStatus("IDLE");
        return dto;
    }

    public Optional<AudiobookDto> update(String id, AudiobookDto incoming) throws ExecutionException, InterruptedException {
        DocumentSnapshot snap = firestore.collection(COLLECTION).document(id).get().get();
        if (!snap.exists()) return Optional.empty();
        AudiobookDto current = fromDoc(snap);
        merge(current, incoming);
        Map<String, Object> data = toMap(current);
        data.put("updatedAt", FieldValue.serverTimestamp());
        firestore.collection(COLLECTION).document(id).update(data).get();
        return Optional.of(current);
    }

    public void deleteById(String id) throws ExecutionException, InterruptedException {
        firestore.collection(COLLECTION).document(id).delete().get();
    }

    public void setProcessingStatus(String id, String status) {
        firestore.collection(COLLECTION).document(id)
                .update("processingStatus", status, "updatedAt", FieldValue.serverTimestamp());
    }

    public void setDistributedTo(String id, String boostyUrl, String telegramFileId, String youtubeVideoId) {
        Map<String, Object> data = new HashMap<>();
        if (boostyUrl != null) data.put("distributedTo.boostyUrl", boostyUrl);
        if (telegramFileId != null) data.put("distributedTo.telegramFileId", telegramFileId);
        if (youtubeVideoId != null) data.put("distributedTo.youtubeVideoId", youtubeVideoId);
        data.put("processingStatus", "DISTRIBUTED");
        data.put("updatedAt", FieldValue.serverTimestamp());
        firestore.collection(COLLECTION).document(id).update(data);
    }

    public void addCoverImageId(String audiobookId, String imageId) {
        firestore.collection(COLLECTION).document(audiobookId)
                .update("coverImageIds", FieldValue.arrayUnion(imageId));
    }

    private static void merge(AudiobookDto current, AudiobookDto incoming) {
        if (incoming.getFileId() != null) current.setFileId(incoming.getFileId());
        if (incoming.getTitle() != null) current.setTitle(incoming.getTitle());
        if (incoming.getAuthor() != null) current.setAuthor(incoming.getAuthor());
        if (incoming.getNarrator() != null) current.setNarrator(incoming.getNarrator());
        if (incoming.getOriginalTitle() != null) current.setOriginalTitle(incoming.getOriginalTitle());
        if (incoming.getPart() != null) current.setPart(incoming.getPart());
        if (incoming.getSourceLink() != null) current.setSourceLink(incoming.getSourceLink());
        if (incoming.getDuration() != null) current.setDuration(incoming.getDuration());
        if (incoming.getUploadDate() != null) current.setUploadDate(incoming.getUploadDate());
        if (incoming.getDescription() != null) current.setDescription(incoming.getDescription());
        if (incoming.getDistributedTo() != null) current.setDistributedTo(incoming.getDistributedTo());
        if (incoming.getProcessingStatus() != null) current.setProcessingStatus(incoming.getProcessingStatus());
    }

    @SuppressWarnings("unchecked")
    private AudiobookDto fromDoc(DocumentSnapshot doc) {
        Map<String, Object> distMap = (Map<String, Object>) doc.get("distributedTo");
        DistributedToDto dist = null;
        if (distMap != null) {
            dist = DistributedToDto.builder()
                    .boostyUrl((String) distMap.get("boostyUrl"))
                    .telegramFileId((String) distMap.get("telegramFileId"))
                    .youtubeVideoId((String) distMap.get("youtubeVideoId"))
                    .build();
        }
        return AudiobookDto.builder()
                .id(doc.getId())
                .fileId(doc.getString("fileId"))
                .title(doc.getString("title"))
                .author(doc.getString("author"))
                .narrator(doc.getString("narrator"))
                .originalTitle(doc.getString("originalTitle"))
                .part(doc.getString("part"))
                .sourceLink(doc.getString("sourceLink"))
                .duration(doc.getLong("duration"))
                .uploadDate(doc.getString("uploadDate"))
                .description(doc.getString("description"))
                .coverImageIds((List<String>) doc.get("coverImageIds"))
                .distributedTo(dist)
                .processingStatus(doc.getString("processingStatus"))
                .build();
    }

    private static Map<String, Object> toMap(AudiobookDto dto) {
        Map<String, Object> m = new HashMap<>();
        m.put("fileId", dto.getFileId());
        m.put("title", dto.getTitle() != null ? dto.getTitle() : "");
        m.put("author", dto.getAuthor());
        m.put("narrator", dto.getNarrator());
        m.put("originalTitle", dto.getOriginalTitle());
        m.put("part", dto.getPart());
        m.put("sourceLink", dto.getSourceLink());
        m.put("duration", dto.getDuration());
        m.put("uploadDate", dto.getUploadDate());
        m.put("description", dto.getDescription());
        m.put("coverImageIds", dto.getCoverImageIds() != null ? dto.getCoverImageIds() : List.of());
        m.put("processingStatus", dto.getProcessingStatus());
        if (dto.getDistributedTo() != null) {
            Map<String, Object> dist = new HashMap<>();
            dist.put("boostyUrl", dto.getDistributedTo().getBoostyUrl());
            dist.put("telegramFileId", dto.getDistributedTo().getTelegramFileId());
            dist.put("youtubeVideoId", dto.getDistributedTo().getYoutubeVideoId());
            m.put("distributedTo", dist);
        } else {
            m.put("distributedTo", Map.of("boostyUrl", "", "telegramFileId", "", "youtubeVideoId", ""));
        }
        return m;
    }
}
```

- [ ] **Step 4: Run — confirm PASS**

```bash
mvn test -Dtest=FirestoreAudiobookServiceTest -q 2>&1 | tail -5
```

- [ ] **Step 5: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/service/FirestoreAudiobookService.java \
        audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/dto/DistributedToDto.java \
        audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/firebase/service/FirestoreAudiobookServiceTest.java
git commit -m "feat: add FirestoreAudiobookService replacing AudiobookLibraryService"
```

---

### Task 8: FirestoreImageService (chunked base64 image storage)

**Files:**
- Create: `firebase/service/FirestoreImageService.java`
- Create: `src/test/java/.../firebase/service/FirestoreImageServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
package kg.automation.rest.automatation.firebase.service;

import com.google.api.core.ApiFutures;
import com.google.cloud.firestore.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.awt.image.BufferedImage;
import java.awt.Color;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class FirestoreImageServiceTest {

    Firestore firestore;
    CollectionReference storeCol;
    CollectionReference chunkCol;
    DocumentReference storeDoc;
    DocumentReference chunkDoc;
    FirestoreImageService service;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() throws Exception {
        firestore = mock(Firestore.class);
        storeCol = mock(CollectionReference.class);
        chunkCol = mock(CollectionReference.class);
        storeDoc = mock(DocumentReference.class);
        chunkDoc = mock(DocumentReference.class);
        when(firestore.collection("image_store")).thenReturn(storeCol);
        when(firestore.collection("image_chunks")).thenReturn(chunkCol);
        when(storeCol.document(anyString())).thenReturn(storeDoc);
        when(chunkCol.document(anyString())).thenReturn(chunkDoc);
        when(storeDoc.set(anyMap())).thenReturn(ApiFutures.immediateFuture(mock(WriteResult.class)));
        when(chunkDoc.set(anyMap())).thenReturn(ApiFutures.immediateFuture(mock(WriteResult.class)));
        service = new FirestoreImageService(firestore);
    }

    @Test
    void storeImage_writesMetadataAndChunk() throws Exception {
        // 100x100 red image — compresses to <10 KB as JPEG, fits in 1 chunk
        BufferedImage img = new BufferedImage(100, 100, BufferedImage.TYPE_INT_RGB);
        var g = img.createGraphics();
        g.setColor(Color.RED);
        g.fillRect(0, 0, 100, 100);
        g.dispose();

        String imageId = service.storeImage(img, "cover", "book-1", 1);

        assertNotNull(imageId);
        verify(storeDoc).set(argThat(m -> "cover".equals(((java.util.Map<?,?>)m).get("type"))));
        verify(chunkDoc, atLeastOnce()).set(argThat(m -> ((java.util.Map<?,?>)m).containsKey("data")));
    }
}
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
mvn test -Dtest=FirestoreImageServiceTest -q 2>&1 | tail -5
```

- [ ] **Step 3: Implement**

```java
package kg.automation.rest.automatation.firebase.service;

import com.google.cloud.firestore.FieldValue;
import com.google.cloud.firestore.Firestore;
import org.springframework.stereotype.Service;

import javax.imageio.IIOImage;
import javax.imageio.ImageIO;
import javax.imageio.ImageWriteParam;
import javax.imageio.ImageWriter;
import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.util.*;
import java.util.concurrent.ExecutionException;

@Service
public class FirestoreImageService {

    private static final String STORE_COLLECTION = "image_store";
    private static final String CHUNKS_COLLECTION = "image_chunks";
    private static final int MAX_DIMENSION = 1400;
    /** Max chars per base64 chunk (~700 KB of string data) */
    private static final int CHUNK_CHARS = 700_000;

    private final Firestore firestore;

    public FirestoreImageService(Firestore firestore) {
        this.firestore = firestore;
    }

    /**
     * Compress {@code image} to JPEG, split into base64 chunks, store in Firestore.
     * @return the imageId (document ID in /image_store)
     */
    public String storeImage(BufferedImage image, String type, String bookId, Integer coverIndex)
            throws IOException, ExecutionException, InterruptedException {
        BufferedImage resized = resizeIfNeeded(image, MAX_DIMENSION);
        byte[] jpegBytes = toJpeg(resized, 0.80f);
        String base64 = Base64.getEncoder().encodeToString(jpegBytes);
        List<String> chunks = splitIntoChunks(base64, CHUNK_CHARS);

        String imageId = UUID.randomUUID().toString();

        // Write metadata document
        Map<String, Object> meta = new HashMap<>();
        meta.put("type", type);
        meta.put("bookId", bookId);
        meta.put("coverIndex", coverIndex);
        meta.put("mimeType", "image/jpeg");
        meta.put("totalChunks", chunks.size());
        meta.put("originalSizeBytes", jpegBytes.length);
        meta.put("createdAt", FieldValue.serverTimestamp());
        firestore.collection(STORE_COLLECTION).document(imageId).set(meta).get();

        // Write chunk documents
        for (int i = 0; i < chunks.size(); i++) {
            Map<String, Object> chunk = new HashMap<>();
            chunk.put("imageId", imageId);
            chunk.put("chunkIndex", i);
            chunk.put("data", chunks.get(i));
            firestore.collection(CHUNKS_COLLECTION).document(imageId + "_" + i).set(chunk).get();
        }
        return imageId;
    }

    /**
     * Reassemble image from Firestore chunks.
     */
    public BufferedImage retrieveImage(String imageId) throws ExecutionException, InterruptedException, IOException {
        var metaSnap = firestore.collection(STORE_COLLECTION).document(imageId).get().get();
        int totalChunks = Objects.requireNonNull(metaSnap.getLong("totalChunks")).intValue();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < totalChunks; i++) {
            var chunkSnap = firestore.collection(CHUNKS_COLLECTION).document(imageId + "_" + i).get().get();
            sb.append(chunkSnap.getString("data"));
        }
        byte[] bytes = Base64.getDecoder().decode(sb.toString());
        return ImageIO.read(new ByteArrayInputStream(bytes));
    }

    private static BufferedImage resizeIfNeeded(BufferedImage img, int maxDim) {
        int w = img.getWidth(), h = img.getHeight();
        if (w <= maxDim && h <= maxDim) return img;
        double scale = (double) maxDim / Math.max(w, h);
        int nw = (int) (w * scale), nh = (int) (h * scale);
        BufferedImage out = new BufferedImage(nw, nh, BufferedImage.TYPE_INT_RGB);
        Graphics2D g = out.createGraphics();
        g.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BICUBIC);
        g.drawImage(img, 0, 0, nw, nh, null);
        g.dispose();
        return out;
    }

    private static byte[] toJpeg(BufferedImage img, float quality) throws IOException {
        // Convert to RGB (strips alpha which JPEG doesn't support)
        BufferedImage rgb = new BufferedImage(img.getWidth(), img.getHeight(), BufferedImage.TYPE_INT_RGB);
        Graphics2D g = rgb.createGraphics();
        g.drawImage(img, 0, 0, null);
        g.dispose();

        ImageWriter writer = ImageIO.getImageWritersByFormatName("jpeg").next();
        ImageWriteParam param = writer.getDefaultWriteParam();
        param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
        param.setCompressionQuality(quality);
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        writer.setOutput(ImageIO.createImageOutputStream(out));
        writer.write(null, new IIOImage(rgb, null, null), param);
        writer.dispose();
        return out.toByteArray();
    }

    private static List<String> splitIntoChunks(String s, int chunkSize) {
        List<String> chunks = new ArrayList<>();
        for (int i = 0; i < s.length(); i += chunkSize) {
            chunks.add(s.substring(i, Math.min(i + chunkSize, s.length())));
        }
        return chunks;
    }
}
```

- [ ] **Step 4: Run — confirm PASS**

```bash
mvn test -Dtest=FirestoreImageServiceTest -q 2>&1 | tail -5
```

- [ ] **Step 5: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/service/FirestoreImageService.java \
        audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/firebase/service/FirestoreImageServiceTest.java
git commit -m "feat: add FirestoreImageService with JPEG compression and chunked base64 storage"
```

---

### Task 9: FirestoreAiLogService

**Files:**
- Create: `firebase/service/FirestoreAiLogService.java`

- [ ] **Step 1: Implement** (no complex test — fire-and-forget logging)

```java
package kg.automation.rest.automatation.firebase.service;

import com.google.cloud.firestore.FieldValue;
import com.google.cloud.firestore.Firestore;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

@Service
public class FirestoreAiLogService {

    private final Firestore firestore;

    public FirestoreAiLogService(Firestore firestore) {
        this.firestore = firestore;
    }

    public void logCall(String provider, String model, String prompt,
                        Integer tokensUsed, Double costUsd, boolean success) {
        Map<String, Object> data = new HashMap<>();
        data.put("provider", provider);
        data.put("model", model);
        data.put("prompt", prompt);
        data.put("tokensUsed", tokensUsed);
        data.put("costUsd", costUsd);
        data.put("success", success);
        data.put("calledAt", FieldValue.serverTimestamp());
        firestore.collection("ai_integration_log").document(UUID.randomUUID().toString()).set(data);
    }

    public String logTrace(String prompt, String imageId) {
        String traceId = UUID.randomUUID().toString();
        Map<String, Object> data = new HashMap<>();
        data.put("prompt", prompt);
        data.put("imageId", imageId != null ? imageId : "");
        data.put("generatedAt", FieldValue.serverTimestamp());
        firestore.collection("ai_traces").document(traceId).set(data);
        return traceId;
    }
}
```

- [ ] **Step 2: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/service/FirestoreAiLogService.java
git commit -m "feat: add FirestoreAiLogService for AI call logging"
```

---

## Phase 3 — Delete Old JPA Code

### Task 10: Delete JPA entities, repositories, and services

- [ ] **Step 1: Delete entity files**

```bash
cd audio-library-automation-bot/src/main/java/kg/automation/rest/automatation

rm -f library/domain/AudiobookEntity.java
rm -f library/domain/AudioDescriptionEntity.java
rm -f library/domain/AiIntegrationLogEntity.java
rm -f library/domain/MediaGroupEntity.java
```

- [ ] **Step 2: Delete repository files**

```bash
rm -f library/repository/AudiobookRepository.java
rm -f library/repository/AudioDescriptionRepository.java
rm -f library/repository/AiIntegrationLogRepository.java
rm -f library/repository/MediaGroupRepository.java
```

- [ ] **Step 3: Delete old service files**

```bash
rm -f library/service/AudiobookLibraryService.java
rm -f library/service/AudioDescriptionPersistenceService.java
rm -f library/service/AiTraceService.java
rm -f library/service/JsonAudiobookImportService.java
rm -f library/service/JsonMediaGroupImportService.java
rm -f file/service/ProperityConfig.java
```

- [ ] **Step 4: Delete obsolete tests**

```bash
cd audio-library-automation-bot/src/test/java/kg/automation/rest/automatation

rm -f library/repository/AudiobookRepositoryTest.java
rm -f library/repository/MediaGroupRepositoryTest.java
rm -f library/service/JsonAudiobookImportServiceTest.java
rm -f library/service/AiTraceServiceTest.java
rm -f library/service/AudioDescriptionPersistenceServiceTest.java
```

- [ ] **Step 5: Commit deletions**

```bash
cd audio-library-automation-bot
git add -A
git commit -m "refactor: delete JPA entities, repositories, and replaced services"
```

---

## Phase 4 — Wire New Services into Existing Code

### Task 11: Update AudiobookV1Controller

**Files:**
- Modify: `library/web/AudiobookV1Controller.java`

- [ ] **Step 1: Replace controller content**

```java
package kg.automation.rest.automatation.library.web;

import kg.automation.rest.automatation.firebase.service.FirestoreAudiobookService;
import kg.automation.rest.automatation.library.dto.AudiobookDto;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.concurrent.ExecutionException;

@RestController
@RequestMapping("/api/v1/library/audiobooks")
@RequiredArgsConstructor
public class AudiobookV1Controller {

    private final FirestoreAudiobookService libraryService;

    @GetMapping
    public List<AudiobookDto> list() throws ExecutionException, InterruptedException {
        return libraryService.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<AudiobookDto> getById(@PathVariable String id) throws ExecutionException, InterruptedException {
        return libraryService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<AudiobookDto> create(@RequestBody AudiobookDto body) throws ExecutionException, InterruptedException {
        AudiobookDto created = libraryService.save(body);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<AudiobookDto> update(@PathVariable String id, @RequestBody AudiobookDto body) throws ExecutionException, InterruptedException {
        return libraryService.update(id, body)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) throws ExecutionException, InterruptedException {
        if (libraryService.findById(id).isEmpty()) return ResponseEntity.notFound().build();
        libraryService.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

Note: `@PathVariable` type changes from `long` to `String` — Firestore IDs are strings.

- [ ] **Step 2: Delete the old controller test and create a new one**

Delete `src/test/java/.../library/web/AudiobookV1ControllerTest.java`, then create a new one:

```java
package kg.automation.rest.automatation.library.web;

import kg.automation.rest.automatation.firebase.service.FirestoreAudiobookService;
import kg.automation.rest.automatation.library.dto.AudiobookDto;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import java.util.List;
import java.util.Optional;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(AudiobookV1Controller.class)
class AudiobookV1ControllerTest {

    @Autowired MockMvc mvc;
    @MockitoBean FirestoreAudiobookService service;

    @Test
    void listReturnsOk() throws Exception {
        when(service.findAll()).thenReturn(List.of(
            AudiobookDto.builder().id("abc").title("Book One").build()
        ));
        mvc.perform(get("/api/v1/library/audiobooks"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$[0].id").value("abc"));
    }

    @Test
    void getByIdReturns404WhenMissing() throws Exception {
        when(service.findById("missing")).thenReturn(Optional.empty());
        mvc.perform(get("/api/v1/library/audiobooks/missing"))
           .andExpect(status().isNotFound());
    }

    @Test
    void createReturns201() throws Exception {
        AudiobookDto dto = AudiobookDto.builder().id("new-id").title("New").processingStatus("IDLE").build();
        when(service.save(any())).thenReturn(dto);
        mvc.perform(post("/api/v1/library/audiobooks")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\":\"New\"}"))
           .andExpect(status().isCreated())
           .andExpect(jsonPath("$.id").value("new-id"));
    }
}
```

- [ ] **Step 3: Run**

```bash
mvn test -Dtest=AudiobookV1ControllerTest -q 2>&1 | tail -5
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 4: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/web/AudiobookV1Controller.java \
        audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/library/web/AudiobookV1ControllerTest.java
git commit -m "feat: wire AudiobookV1Controller to FirestoreAudiobookService"
```

---

### Task 12: Update AudioConcatenatorService — replace JPA with job log

**Files:**
- Modify: `audio/service/AudioConcatenatorService.java`

- [ ] **Step 1: Replace the `audioDescriptionPersistenceService` dependency**

In `AudioConcatenatorService.java`, replace:
```java
// Remove this field
@Autowired
private AudioDescriptionPersistenceService audioDescriptionPersistenceService;
```

Add:
```java
@Autowired
private FirestoreJobLogService jobLogService;
```

Add import:
```java
import kg.automation.rest.automatation.firebase.service.FirestoreJobLogService;
import java.util.UUID;
```

Replace the `audioDescriptionPersistenceService.upsert(...)` call block inside `concatenateAudio()`:

```java
// Replace this block:
AudioDescription audioDescription = new AudioDescription();
audioDescription.setDirectory(inputDir);
audioDescription.setCountFiles(audios.length);
audioDescription.setAudioConcatenateDirectoryPath(outputDir);
audioDescription.setAudioConcatenateFileAbsolutePath(outputDir+part+".mp3");
audioDescription.setCyclePart(part);
audioDescriptionPersistenceService.upsert(audioDescription, inputDir);

// With this:
String jobId = UUID.randomUUID().toString();
jobLogService.startJob(jobId, "CONCATENATE", null);
jobLogService.completeJob(jobId, "Concatenated " + audios.length + " files → " + outputDir + part + ".mp3");
```

- [ ] **Step 2: Compile check**

```bash
mvn compile -pl audio-library-automation-bot -q 2>&1 | grep "error:" | head -20
```

Expected: no errors in `AudioConcatenatorService`.

- [ ] **Step 3: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/audio/service/AudioConcatenatorService.java
git commit -m "refactor: replace AudioDescriptionPersistenceService with FirestoreJobLogService in concatenator"
```

---

### Task 13: Update AudioCompressorService — add job logging

**Files:**
- Modify: `audio/service/AudioCompressorService.java`

- [ ] **Step 1: Inject FirestoreJobLogService and wrap compressAudio**

Add field and import to `AudioCompressorService`:

```java
import kg.automation.rest.automatation.firebase.service.FirestoreJobLogService;
import java.util.UUID;

@Autowired
private FirestoreJobLogService jobLogService;
```

Wrap the body of `compressAudio()` with job log calls:

```java
public Response compressAudio(String inputDir, String outputDir) {
    String jobId = UUID.randomUUID().toString();
    jobLogService.startJob(jobId, "COMPRESS", null);
    Response response = new Response();
    try {
        // ... existing try block contents unchanged ...
        // After the existing response.setStatus("success") line, add:
        jobLogService.completeJob(jobId, response.getMessage());
    } catch (IOException | InterruptedException e) {
        response.setStatus("error");
        response.setMessage("Error compressing audio files: " + e.getMessage());
        jobLogService.failJob(jobId, response.getMessage(), e.toString());
    }
    return response;
}
```

- [ ] **Step 2: Compile check**

```bash
mvn compile -pl audio-library-automation-bot -q 2>&1 | grep "error:"
```

- [ ] **Step 3: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/audio/service/AudioCompressorService.java
git commit -m "feat: add FirestoreJobLogService job tracking to AudioCompressorService"
```

---

### Task 14: Update BookCoverService — store covers in Firestore after generation

**Files:**
- Modify: `image/service/BookCoverService.java`

- [ ] **Step 1: Inject FirestoreImageService and FirestoreJobLogService**

Add to `BookCoverService`:

```java
import kg.automation.rest.automatation.firebase.service.FirestoreImageService;
import kg.automation.rest.automatation.firebase.service.FirestoreJobLogService;
import org.springframework.beans.factory.annotation.Autowired;
import java.util.UUID;

@Autowired
private FirestoreImageService firestoreImageService;

@Autowired
private FirestoreJobLogService jobLogService;
```

- [ ] **Step 2: Update `generateCovers()` to store in Firestore**

Replace `generateCovers()`:

```java
public Response generateCovers(String inputImage, String outputDir, String title, int number) {
    String jobId = UUID.randomUUID().toString();
    jobLogService.startJob(jobId, "COVER_GENERATE", null);
    Response response = new Response();
    try {
        for (int i = 1; i <= number; i++) {
            BufferedImage cover = createCoverImage(inputImage, title, i);
            // Write to temp dir for Boosty/distribution use
            File outFile = new File(outputDir + "/" + i + ".png");
            ImageIO.write(cover, "png", outFile);
            // Also store in Firestore as chunked base64
            firestoreImageService.storeImage(cover, "cover", title, i);
        }
        response.setStatus("success");
        response.setMessage("Book covers created successfully.");
        jobLogService.completeJob(jobId, response.getMessage());
    } catch (Exception e) {
        response.setStatus("error");
        response.setMessage("Error creating book cover: " + e.getMessage());
        jobLogService.failJob(jobId, response.getMessage(), e.toString());
    }
    return response;
}
```

- [ ] **Step 3: Extract `createCoverImage()` helper from the existing `createCover()` private method**

Rename existing private `createCover(String inputImage, String outputDir, String title, int number)` to `createCoverImage(String inputImage, String title, int number)` returning `BufferedImage`:

```java
private BufferedImage createCoverImage(String inputImage, String title, int number) throws Exception {
    BufferedImage image = ImageIO.read(new File(inputImage));
    Graphics2D g2d = image.createGraphics();
    g2d.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);

    g2d.setFont(new Font("Comic Sans MS", Font.BOLD, 70));
    g2d.setColor(Color.RED);
    FontMetrics fm = g2d.getFontMetrics();
    String audioBookText = "АУДИОКНИГА";
    int audioX = (image.getWidth() - fm.stringWidth(audioBookText)) / 2;
    g2d.drawString(audioBookText, audioX, 100);

    g2d.setFont(new Font("Comic Sans MS", Font.BOLD, 80));
    fm = g2d.getFontMetrics();
    String titleText = title + " " + number;
    int titleX = image.getWidth() / 2 - fm.stringWidth(titleText) / 2;
    g2d.drawString(titleText, titleX, image.getHeight() - 300);

    g2d.setFont(new Font("Comic Sans MS", Font.BOLD, 96));
    fm = g2d.getFontMetrics();
    String footerText = "~~~~~~~~~~2024~~~~~~~~~~";
    int footerX = image.getWidth() / 2 - fm.stringWidth(footerText) / 2;
    g2d.drawString(footerText, footerX, image.getHeight() - 50);

    g2d.dispose();
    return image;
}
```

- [ ] **Step 4: Compile check**

```bash
mvn compile -pl audio-library-automation-bot -q 2>&1 | grep "error:"
```

- [ ] **Step 5: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/image/service/BookCoverService.java
git commit -m "feat: BookCoverService stores generated covers in Firestore via FirestoreImageService"
```

---

### Task 15: Fix hardcoded API key in RestCallService

**Files:**
- Modify: `gateway/service/RestCallService.java` (or wherever the OpenAI/search calls live)

- [ ] **Step 1: Find and replace hardcoded key**

```bash
grep -rn "Bearer\|api.key\|sk-\|Authorization" \
  audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/gateway/ \
  audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/search/
```

- [ ] **Step 2: Inject from properties**

In each service class that has a hardcoded key, add `@Value` injection:

```java
import org.springframework.beans.factory.annotation.Value;

@Value("${openai.api.key:}")
private String openAiApiKey;

@Value("${search.api.token:}")
private String searchApiToken;

@Value("${search.api.url:http://localhost:8080}")
private String searchApiUrl;
```

Replace hardcoded string literals in `HttpHeaders` setup:
```java
// Before:
headers.setBearerAuth("sk-hardcoded-key-here");

// After:
headers.setBearerAuth(openAiApiKey);
```

- [ ] **Step 3: Compile check**

```bash
mvn compile -pl audio-library-automation-bot -q 2>&1 | grep "error:"
```

- [ ] **Step 4: Commit**

```bash
git add audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/gateway/ \
        audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/search/
git commit -m "fix: move hardcoded API keys to @Value injection from application.properties"
```

---

### Task 16: Full compile — fix remaining broken imports

At this point the major classes are in place. Run a full compile and fix remaining errors.

- [ ] **Step 1: Full compile**

```bash
cd audio-library-automation-bot
mvn compile -q 2>&1 | grep "error:" | head -50
```

- [ ] **Step 2: Fix each error**

Common patterns you'll see and how to fix them:
- Any remaining `import .../ProperityConfig` → delete that import, the caller no longer needs path config
- Any `AudiobookLibraryService` reference → replace with `FirestoreAudiobookService`
- Any `AiTraceService` reference → replace with `FirestoreAiLogService`
- Any `AudioDescriptionPersistenceService` reference → replace with `FirestoreJobLogService`
- Any JPA `@Transactional` that now has no JPA context → remove the annotation from that method

- [ ] **Step 3: Compile again — confirm zero errors**

```bash
mvn compile -q 2>&1 | grep "error:"
```

Expected: no output.

- [ ] **Step 4: Commit all fixes**

```bash
git add -A
git commit -m "fix: resolve all remaining broken imports after JPA removal"
```

---

## Phase 5 — Frontend Firebase Integration

### Task 17: Add Firebase SDK to frontend

**Files:**
- Modify: `audio-frontend/package.json`
- Create: `audio-frontend/src/firebase.ts`
- Modify: `audio-frontend/.env` (or create `.env.local`)

- [ ] **Step 1: Install firebase**

```bash
cd audio-frontend
npm install firebase
```

- [ ] **Step 2: Create src/firebase.ts**

```typescript
import { initializeApp } from 'firebase/app';
import { getFirestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId: import.meta.env.VITE_FIREBASE_APP_ID,
};

export const app = initializeApp(firebaseConfig);
export const db = getFirestore(app);
```

- [ ] **Step 3: Add env vars**

Create `audio-frontend/.env.local` (not committed):

```
VITE_FIREBASE_API_KEY=your-api-key-from-firebase-console
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=audio-library-system
VITE_FIREBASE_STORAGE_BUCKET=audio-library-system.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=your-sender-id
VITE_FIREBASE_APP_ID=your-app-id
```

Add `.env.local` to `.gitignore` if not already there.

- [ ] **Step 4: Commit**

```bash
cd audio-frontend
git add src/firebase.ts package.json package-lock.json
git commit -m "feat: add Firebase JS SDK and Firestore client initialization"
```

---

### Task 18: Add live job log panel to frontend

**Files:**
- Modify: `audio-frontend/src/api/client.ts`
- Modify: `audio-frontend/src/App.tsx`

- [ ] **Step 1: Add Firestore job log hook to client.ts**

Add to `audio-frontend/src/api/client.ts`:

```typescript
import { db } from '../firebase';
import { collection, query, orderBy, limit, onSnapshot } from 'firebase/firestore';

export interface JobLog {
  id: string;
  type: string;
  status: 'STARTED' | 'COMPLETED' | 'FAILED';
  audiobookId: string;
  message: string;
  errorDetail: string | null;
  startedAt: { seconds: number } | null;
  finishedAt: { seconds: number } | null;
}

export function subscribeToJobLogs(
  callback: (logs: JobLog[]) => void
): () => void {
  const q = query(
    collection(db, 'job_logs'),
    orderBy('startedAt', 'desc'),
    limit(20)
  );
  return onSnapshot(q, (snap) => {
    const logs: JobLog[] = snap.docs.map((doc) => ({
      id: doc.id,
      ...(doc.data() as Omit<JobLog, 'id'>),
    }));
    callback(logs);
  });
}
```

- [ ] **Step 2: Add job log panel to App.tsx**

In `App.tsx`, add state and effect:

```tsx
import { subscribeToJobLogs, JobLog } from './api/client';
import { useEffect, useState } from 'react';

// Inside the App component:
const [jobLogs, setJobLogs] = useState<JobLog[]>([]);

useEffect(() => {
  const unsubscribe = subscribeToJobLogs(setJobLogs);
  return unsubscribe;
}, []);
```

Add the job log panel to the JSX (place below the existing health check panels):

```tsx
<section className="mt-6">
  <h2 className="text-lg font-semibold mb-2">Processing Jobs</h2>
  <div className="space-y-2">
    {jobLogs.length === 0 && (
      <p className="text-gray-500 text-sm">No jobs yet.</p>
    )}
    {jobLogs.map((job) => (
      <div
        key={job.id}
        className={`rounded p-3 text-sm border ${
          job.status === 'COMPLETED' ? 'border-green-600 bg-green-950' :
          job.status === 'FAILED'    ? 'border-red-600 bg-red-950' :
                                       'border-yellow-600 bg-yellow-950'
        }`}
      >
        <span className="font-mono font-bold">{job.type}</span>
        {' · '}
        <span>{job.status}</span>
        {job.message && <span className="ml-2 text-gray-400">{job.message}</span>}
        {job.errorDetail && (
          <p className="mt-1 text-red-400 text-xs">{job.errorDetail}</p>
        )}
      </div>
    ))}
  </div>
</section>
```

- [ ] **Step 3: Build check**

```bash
cd audio-frontend
npm run build 2>&1 | tail -10
```

Expected: successful build with no TypeScript errors.

- [ ] **Step 4: Commit**

```bash
git add audio-frontend/src/api/client.ts audio-frontend/src/App.tsx
git commit -m "feat: add real-time Firestore job log panel to frontend dashboard"
```

---

## Phase 6 — One-Time Data Migration

### Task 19: Write migration script

**Files:**
- Create: `scripts/migrate-to-firestore.py`

- [ ] **Step 1: Export PostgreSQL data to JSON**

```bash
# Run these against your PostgreSQL instance before shutting it down
psql -U audio -d audio_library -c "\copy (SELECT * FROM audiobook) TO 'audiobooks.json' WITH (FORMAT json)"
psql -U audio -d audio_library -c "\copy (SELECT * FROM parsed_book) TO 'parsed_books.json' WITH (FORMAT json)"
psql -U audio -d audio_library -c "\copy (SELECT * FROM media_group) TO 'media_groups.json' WITH (FORMAT json)"
```

- [ ] **Step 2: Create the migration script**

```python
#!/usr/bin/env python3
"""
One-time migration: PostgreSQL JSON exports → Firestore.
Usage:
  pip install firebase-admin
  export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
  python scripts/migrate-to-firestore.py
"""
import json
import sys
import os
import firebase_admin
from firebase_admin import credentials, firestore

def load_json(path):
    if not os.path.exists(path):
        print(f"  SKIP: {path} not found")
        return []
    with open(path) as f:
        # psql JSON output is one object per line
        return [json.loads(line) for line in f if line.strip()]

def migrate():
    cred = credentials.ApplicationDefault()
    firebase_admin.initialize_app(cred)
    db = firestore.client()

    # Migrate audiobooks
    audiobooks = load_json('audiobooks.json')
    print(f"Migrating {len(audiobooks)} audiobooks...")
    for book in audiobooks:
        doc_id = str(book.get('id', ''))
        data = {
            'fileId': book.get('file_id'),
            'title': book.get('title', ''),
            'author': book.get('author'),
            'narrator': book.get('narrator'),
            'originalTitle': book.get('original_title'),
            'part': book.get('part'),
            'sourceLink': book.get('source_link'),
            'duration': book.get('duration'),
            'uploadDate': str(book.get('upload_date', '')),
            'description': book.get('description'),
            'coverImageIds': [],
            'distributedTo': {
                'boostyUrl': None,
                'telegramFileId': book.get('file_id'),  # preserve telegram file id
                'youtubeVideoId': None,
            },
            'processingStatus': 'DISTRIBUTED' if book.get('file_id') else 'IDLE',
            'createdAt': firestore.SERVER_TIMESTAMP,
            'updatedAt': firestore.SERVER_TIMESTAMP,
        }
        db.collection('audiobooks').document(doc_id).set(data)
        print(f"  ✓ {book.get('title', doc_id)}")

    # Migrate parsed_books
    parsed = load_json('parsed_books.json')
    print(f"Migrating {len(parsed)} parsed books...")
    for book in parsed:
        doc_id = str(book.get('id', ''))
        data = {
            'title': book.get('title'),
            'description': book.get('description'),
            'sourceDirectory': book.get('source_directory'),
            'url': book.get('source_link', ''),
            'createdAt': firestore.SERVER_TIMESTAMP,
        }
        db.collection('parsed_books').document(doc_id).set(data)
        print(f"  ✓ {book.get('title', doc_id)}")

    # Migrate media_groups
    groups = load_json('media_groups.json')
    print(f"Migrating {len(groups)} media groups...")
    for group in groups:
        doc_id = str(group.get('id', ''))
        data = {
            'name': group.get('name', ''),
            'items': group.get('items', []),
            'createdAt': firestore.SERVER_TIMESTAMP,
        }
        db.collection('media_groups').document(doc_id).set(data)
        print(f"  ✓ {group.get('name', doc_id)}")

    print("\nMigration complete.")

if __name__ == '__main__':
    migrate()
```

- [ ] **Step 3: Commit**

```bash
git add scripts/migrate-to-firestore.py
git commit -m "feat: add one-shot PostgreSQL → Firestore migration script"
```

---

### Task 20: Run migration and decommission PostgreSQL

- [ ] **Step 1: Verify Firebase project is set up**

In the Firebase console:
- Project: `audio-library-system` (from `.firebaserc`)
- Firestore: enabled in Native Mode
- `GOOGLE_APPLICATION_CREDENTIALS` set to your service account JSON path

- [ ] **Step 2: Export PostgreSQL data**

```bash
psql -U audio -d audio_library -c "\copy (SELECT * FROM audiobook) TO 'audiobooks.json' WITH (FORMAT json)"
psql -U audio -d audio_library -c "\copy (SELECT * FROM parsed_book) TO 'parsed_books.json' WITH (FORMAT json)"
psql -U audio -d audio_library -c "\copy (SELECT * FROM media_group) TO 'media_groups.json' WITH (FORMAT json)"
```

- [ ] **Step 3: Run migration**

```bash
pip install firebase-admin
python scripts/migrate-to-firestore.py
```

Expected: all books, parsed books, and media groups printed with ✓.

- [ ] **Step 4: Verify in Firebase console**

Open the Firestore data viewer in Firebase console. Confirm `/audiobooks` collection shows the migrated documents.

- [ ] **Step 5: Start the backend and verify it boots**

```bash
cd audio-library-automation-bot
mvn spring-boot:run
```

Expected: no `UnsatisfiedDependencyException`, no `DataSourceProperties` errors. Backend starts on port 8088.

- [ ] **Step 6: Smoke test the API**

```bash
curl -s http://localhost:8088/api/v1/library/audiobooks | head -c 200
```

Expected: JSON array of audiobooks from Firestore.

```bash
curl -s http://localhost:8088/actuator/health
```

Expected: `{"status":"UP"}`

- [ ] **Step 7: Test cover generation writes to Firestore**

```bash
curl -s -X POST http://localhost:8088/api/book-cover/generate \
  -H "Content-Type: application/json" \
  -d '{"imagePath":"/path/to/test.jpg","title":"TestBook","count":1}'
```

Open Firestore console → `/image_store` — should see a new document with `type: "cover"`.

- [ ] **Step 8: Verify job logs appear in frontend**

Start frontend: `cd audio-frontend && npm run dev`

Trigger a compress job:
```bash
curl -X POST http://localhost:8088/api/audio/compress \
  -H "Content-Type: application/json" \
  -d '{"inputDirectory":"/tmp/test","outputDirectory":"/tmp/out"}'
```

Open `http://localhost:5173` — the "Processing Jobs" panel should show the COMPRESS job updating in real time.

- [ ] **Step 9: Decommission PostgreSQL**

Only after confirming everything above works:

```bash
# Stop PostgreSQL service (Windows)
net stop postgresql-x64-15

# Optionally delete local data dirs (backup first)
# rm -rf H:/Projects/Audio-library-System/audio-library-automation-bot/data/audio
# rm -rf H:/Projects/Audio-library-System/audio-library-automation-bot/data/image
# rm -rf H:/Projects/Audio-library-System/audio-library-automation-bot/data/video
```

- [ ] **Step 10: Final commit**

```bash
git add -A
git commit -m "feat: complete Firebase migration — PostgreSQL decommissioned, all data in Firestore"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task(s) |
|-----------------|---------|
| Remove PostgreSQL/JPA/Flyway deps | Task 1 |
| `processing.temp.dir` property | Task 2 |
| Firebase always enabled | Task 3 |
| `ProcessingTempDirService` | Task 4 |
| `FirestoreJobLogService` | Task 5 |
| `AudiobookDto` updated with new fields | Task 6 |
| `FirestoreAudiobookService` | Task 7 |
| `FirestoreImageService` chunked base64 | Task 8 |
| `FirestoreAiLogService` | Task 9 |
| Delete JPA classes | Task 10 |
| `AudiobookV1Controller` → Firestore | Task 11 |
| Audio services use job logging | Tasks 12–13 |
| `BookCoverService` → Firestore images | Task 14 |
| API key security fix | Task 15 |
| Compile — fix remaining errors | Task 16 |
| Frontend Firebase SDK | Task 17 |
| Live job log panel | Task 18 |
| Migration script | Task 19 |
| Execute migration + decommission | Task 20 |
| `DistributedToDto` + `setDistributedTo()` | Task 7, Task 6 |
| JPEG compress + resize before chunking | Task 8 |
| Image reassembly | Task 8 |

All spec requirements covered.

**Type consistency check:**
- `AudiobookDto.id` is `String` throughout (Tasks 6, 7, 11)
- `FirestoreJobLogService.startJob(jobId, type, audiobookId)` — 3-arg signature used in Tasks 5, 12, 13, 14
- `FirestoreImageService.storeImage(BufferedImage, type, bookId, coverIndex)` — 4-arg signature used in Tasks 8, 14
- `ProcessingTempDirService.createJobDir(jobId)` / `cleanupJobDir(jobId)` — consistent in Tasks 4 usage

**No placeholders:** All tasks have complete code, no "implement later" or "similar to above" patterns.
