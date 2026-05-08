# Application / database alignment implementation plan

> **For agentic workers:** Use **subagent-driven-development** (recommended) or **executing-plans** to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align Java, REST, Firestore mirror, frontend copy/types, and distribution persistence with the PostgreSQL schema approved in [`2026-05-07-application-db-alignment-design.md`](../specs/2026-05-07-application-db-alignment-design.md) (option **B**, dual-write distribution).

**Architecture:** Map nullable catalog FKs; add `book_chapter` REST with **PUT upsert**; introduce `DistributionPersistenceService` for `distribution` / `distribution_result`; call it when catalog distribution cache updates and `cycle_id` is known; audit `job_queue` population for existing executors; fix stale names in comments and frontend.

**Tech stack:** Java 17, Spring Boot 3.4, JPA, Flyway (no new migrations unless schema bug), React/Vite frontend, Vitest optional for TS types.

**Verify:** From `audio-library-automation-bot/`, run `.\mvnw.cmd test`. From `audio-frontend/`, run `npm run lint`.

---

## File map (expected touch list)

| Area | Create | Modify |
|------|--------|--------|
| Catalog | — | `LibraryAudiobookEntity.java`, `AudiobookDto.java`, `LibraryAudiobookService.java`, `FirestoreAudiobookService.java`, `AudioLibraryService.java` |
| Catalog tests | `LibraryAudiobookServiceTest.java`, extend `AudiobookV1ControllerTest.java` | — |
| Chapters | `BookChapterEntity.java`, `BookChapterRepository.java`, `BookChapterService.java`, `BookChapterResponse.java`, `BookChapterUpsertRequest.java` | `CycleV1Controller.java` |
| Chapters tests | `BookChapterRepositoryIT.java` (or `BookChapterApiIT.java`) | — |
| Distribution | `DistributionPersistenceService.java` | `LibraryAudiobookService.java` |
| Distribution tests | `DistributionPersistenceServiceIT.java` | — |
| Jobs | — | `ConcatenateProcessingJobRunner.java` (comment/adjust if needed) |
| Hygiene | — | `BookCover.java`, `BookCoverGenerateResponse.java`, `BookCoverGeneratedPart.java`, `audio-frontend/src/api/client.ts`, `audio-frontend/src/App.tsx` |

---

### Task 1: Catalog FK mapping (`book_id`, `cycle_id`)

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/domain/LibraryAudiobookEntity.java`
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/dto/AudiobookDto.java`
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/service/LibraryAudiobookService.java`

- [ ] **Step 1:** Add nullable columns to `LibraryAudiobookEntity`:

```java
    @Column(name = "book_id")
    private Long bookId;

    @Column(name = "cycle_id")
    private Long cycleId;
```

- [ ] **Step 2:** Add fields to `AudiobookDto` (same package, Lombok will generate getters/setters):

```java
    /** Internal FK to {@code book.id}; optional. */
    private Long bookId;
    /** Internal FK to {@code cycle.id}; optional. */
    private Long cycleId;
```

- [ ] **Step 3:** In `LibraryAudiobookService.mergeIncomingOntoEntity`, after `coverImageIds` block and before `distributedTo`:

```java
        if (incoming.getBookId() != null) {
            target.setBookId(incoming.getBookId());
        }
        if (incoming.getCycleId() != null) {
            target.setCycleId(incoming.getCycleId());
        }
```

- [ ] **Step 4:** In `toDto`, add to the builder chain after `fileId`:

```java
                .bookId(e.getBookId())
                .cycleId(e.getCycleId())
```

- [ ] **Step 5:** Run `.\mvnw.cmd test -q` — expect pass (Hibernate validate against real Postgres in tests that start the full context).

---

### Task 2: Firestore mirror parity

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/firebase/service/FirestoreAudiobookService.java`

- [ ] **Step 1:** In `merge`, mirror catalog fields:

```java
        if (incoming.getBookId() != null) current.setBookId(incoming.getBookId());
        if (incoming.getCycleId() != null) current.setCycleId(incoming.getCycleId());
```

- [ ] **Step 2:** In `fromDoc`, read Long-safe values (Firestore may store numbers):

```java
        Long bookId = doc.contains("bookId") ? doc.getLong("bookId") : null;
        Long cycleId = doc.contains("cycleId") ? doc.getLong("cycleId") : null;
```

Add `.bookId(bookId).cycleId(cycleId)` to the `AudiobookDto.builder()` chain.

- [ ] **Step 3:** In `toMap`, put keys when non-null:

```java
        if (dto.getBookId() != null) m.put("bookId", dto.getBookId());
        if (dto.getCycleId() != null) m.put("cycleId", dto.getCycleId());
```

- [ ] **Step 4:** Run `.\mvnw.cmd test -q`.

---

### Task 3: Telegram `AudioLibraryService` DTO parity

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/telegram/service/AudioLibraryService.java`

- [ ] **Step 1:** Add to inner class `Audiobook`: `public Long bookId;` and `public Long cycleId;` (and optionally `public DistributedToDto distributedTo` if bot should round-trip distribution — if added, import DTO and map in `toLibraryDto` / `fromLibraryDto`).

- [ ] **Step 2:** Extend `toLibraryDto` builder with `.bookId(a.bookId).cycleId(a.cycleId)` (and `distributedTo` if introduced).

- [ ] **Step 3:** Extend `fromLibraryDto` to copy `bookId` / `cycleId` onto `Audiobook`.

- [ ] **Step 4:** Run `.\mvnw.cmd test -q`.

---

### Task 4: `book_chapter` persistence + API (PUT upsert)

**Files:**
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/domain/BookChapterEntity.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/repository/BookChapterRepository.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/service/BookChapterService.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/dto/BookChapterResponse.java`
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/dto/BookChapterUpsertRequest.java`
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/web/CycleV1Controller.java`

- [ ] **Step 1:** Implement `BookChapterEntity` mapping `book_chapter` (table name `"book_chapter"`), columns: `id`, `cycle_id`, `chapter_number`, `chapter_title`, `chapter_text`, `created_at` (read-only after insert; use `@PrePersist` for `createdAt` if missing). No `updated_at` column in DB — do not map it.

- [ ] **Step 2:** `BookChapterRepository`:

```java
public interface BookChapterRepository extends JpaRepository<BookChapterEntity, Long> {
    List<BookChapterEntity> findByCycleIdOrderByChapterNumberAsc(Long cycleId);
    Optional<BookChapterEntity> findByCycleIdAndChapterNumber(Long cycleId, Integer chapterNumber);
}
```

- [ ] **Step 3:** `BookChapterService` (`@Service`, `@RequiredArgsConstructor`): inject `BookChapterRepository`, `CycleRepository`. Methods: `listChapters(Long cycleId)` returns empty list if cycle missing; `upsertChapter(Long cycleId, int chapterNumber, BookChapterUpsertRequest req)` — find existing by `(cycleId, chapterNumber)` or create new, set title/text from request, save.

- [ ] **Step 4:** DTOs: `BookChapterResponse` with `id`, `cycleId`, `chapterNumber`, `chapterTitle`, `chapterText`, `createdAt`; `BookChapterUpsertRequest` with `chapterTitle`, `chapterText` (both optional).

- [ ] **Step 5:** Add to `CycleV1Controller` endpoints:

```java
    @GetMapping("/{cycleId}/chapters")
    public ResponseEntity<List<BookChapterResponse>> listChapters(@PathVariable Long cycleId) { ... }

    @PutMapping("/{cycleId}/chapters/{chapterNumber}")
    public ResponseEntity<BookChapterResponse> upsertChapter(
            @PathVariable Long cycleId,
            @PathVariable int chapterNumber,
            @RequestBody BookChapterUpsertRequest body) { ... }
```

Return **404** when `!cycleRepository.existsById(cycleId)` for both.

- [ ] **Step 6:** Add integration test `BookChapterRepositoryIT` in `src/test/java/.../integration/`: use the same **embedded PostgreSQL + Flyway** pattern as `FlywayCycleSchemaIT` (`EmbeddedPostgres.builder().start()`, migrate, JDBC insert into `book` then `cycle` with valid `book_id`, then save chapter via repository or service, assert `findByCycleIdOrderByChapterNumberAsc` ordering). Alternatively `@SpringBootTest` + test profile if already standard in repo.

- [ ] **Step 7:** Run `.\mvnw.cmd test -q`.

---

### Task 5: `DistributionPersistenceService`

**Files:**
- Create: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/pipeline/service/DistributionPersistenceService.java`
- Create: `audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/pipeline/service/DistributionPersistenceServiceIT.java` (or extend existing integration harness)

- [ ] **Step 1:** Implement service with constructor injection: `PlatformRepository`, `DistributionRepository`, `DistributionResultRepository`. Method signature:

```java
@Transactional
public void recordPlatformResult(
        Long cycleId,
        Long audioPartIdOrNull,
        String platformCode,
        PlatformResultStatus status,
        String platformPostId,
        String url,
        Map<String, Object> platformMetadata) {
```

- [ ] **Step 2:** Body logic:
  1. `PlatformEntity platform = platformRepository.findByCode(platformCode).orElseThrow(() -> new IllegalStateException("platform not seeded: " + platformCode));`
  2. Resolve `DistributionEntity dist`: if `audioPartIdOrNull == null`, `distributionRepository.findByCycleIdAndAudioPartIdIsNull(cycleId)`; else `findByCycleIdAndAudioPartId(cycleId, audioPartIdOrNull)`. If empty, create new `DistributionEntity`, set `cycleId`, `audioPartId` (nullable), `status = ReleaseStatus.PENDING`, save and flush to obtain id.
  3. Load `Optional<DistributionResultEntity> existing = distributionResultRepository.findByDistributionIdAndPlatformId(dist.getId(), platform.getId())`. If present, update; else new entity with `distributionId`, `platformId`. Set `status`, `platformPostId`, `url`, `lastError` (if FAILED), `platformMetadata` (non-null map), `distributedAt = Instant.now()` when `status == PlatformResultStatus.SUCCESS`. Save.

- [ ] **Step 3:** Use `platformCode` values **`YOUTUBE`**, **`TELEGRAM`**, **`BOOSTY`** — must match `V202605072450__platform.sql` (uppercase).

- [ ] **Step 4:** Integration test: embedded Postgres + Flyway, seed platforms already in migration, insert `book` + `cycle`, call `recordPlatformResult`, assert one row in `distribution` and `distribution_result` via `JdbcTemplate` or repository.

- [ ] **Step 5:** Run `.\mvnw.cmd test -q`.

---

### Task 6: Wire dual-write from catalog updates

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/library/service/LibraryAudiobookService.java`

- [ ] **Step 1:** Inject `DistributionPersistenceService` via constructor (`@RequiredArgsConstructor` adds field). If optional for tests, use `@Autowired(required = false)` only if tests use a slice without the bean — prefer always-on bean in main context.

- [ ] **Step 2:** At **end** of `mergeIncomingOntoEntity` (after all field copies), if `target.getCycleId() != null` and `incoming.getDistributedTo() != null`, invoke:
  - When `incoming.getDistributedTo().getYoutubeVideoId()` non-blank: `recordPlatformResult(target.getCycleId(), null, "YOUTUBE", PlatformResultStatus.SUCCESS, youtubeId.trim(), "https://www.youtube.com/watch?v=" + youtubeId.trim(), Map.of("video_id", youtubeId.trim()))` (adjust URL scheme if storing full URLs only).
  - When `incoming.getDistributedTo().getTelegramFileId()` non-blank: `recordPlatformResult(target.getCycleId(), null, "TELEGRAM", PlatformResultStatus.SUCCESS, fileId.trim(), null, Map.of("file_id", fileId.trim()))`.

Use `PlatformResultStatus.SUCCESS`. For partial failures (future), use `FAILED` with `last_error` — out of scope unless REST gains failure payload.

- [ ] **Step 3:** Guard: if both YouTube and Telegram updated in same request, call service twice (two platform rows).

- [ ] **Step 4:** Run `.\mvnw.cmd test -q`.

---

### Task 7: Upload / poster hooks (incremental)

**Files:**
- Modify (as applicable): `poster/youtube/controller/YoutubeController.java`, `poster/boosty/controller/BoostyPosterController.java`, `telegram/component/AudioLibraryBot.java`

- [ ] **Step 1:** `YouTubeUploader.upload` currently throws stub — add **JavaDoc** only: when implementation returns a video id, call `DistributionPersistenceService.recordPlatformResult` with `YOUTUBE` (do not block stub on full OAuth).

- [ ] **Step 2:** `Poster` / `BoostyPosterController`: when a Boosty post id becomes available in code, add the same for `BOOSTY`. If not available, document in class-level JavaDoc the gap (aligns with spec “enumerate call sites”).

- [ ] **Step 3:** If `AudioLibraryBot` persists `AudiobookDto` with `distributedTo` after channel publish, ensure **Task 3** maps `distributedTo` so **Task 6** fires — implement minimal mapping if bot sets telegram file id on DTO today.

- [ ] **Step 4:** Run `.\mvnw.cmd test -q`.

---

### Task 8: `job_queue` FK audit (`AUDIO_CONCATENATE`)

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/jobs/service/ConcatenateProcessingJobRunner.java` (comments only unless code change justified)

- [ ] **Step 1:** Confirm `ProcessingJobService` sets `cycle_id` and `audio_part_id` on submit — already true.

- [ ] **Step 2:** Add class-level comment: `video_part_id`, `image_asset_id`, `ai_content_id`, `distribution_id` are **N/A** for `AUDIO_CONCATENATE`; other job types must set per ER plan §2.7 when executors exist.

- [ ] **Step 3:** No code change unless missing `completed_at` — `ConcatenateProcessingJobRunner` already sets `started_at` / `completed_at`.

- [ ] **Step 4:** Run `.\mvnw.cmd test -q`.

---

### Task 9: Legacy naming hygiene (Java + frontend)

**Files:**
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/image/BookCover.java`
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/image/BookCoverGenerateResponse.java`
- Modify: `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/image/BookCoverGeneratedPart.java`
- Modify: `audio-frontend/src/api/client.ts`
- Modify: `audio-frontend/src/App.tsx`

- [ ] **Step 1:** Replace Javadoc mentions of **`image_parts`** with **`image_asset`** (IDs refer to `image_asset.id`).

- [ ] **Step 2:** In `client.ts`, change `AudiobookRow` comment to `library_audiobook` and add optional fields:

```typescript
  bookId?: number;
  cycleId?: number;
```

- [ ] **Step 3:** In `App.tsx`, replace visible copy `library_audiobooks` with `library_audiobook` if shown to users.

- [ ] **Step 4:** Run `cd audio-library-automation-bot && .\mvnw.cmd test -q` and `cd audio-frontend && npm run lint`.

---

### Task 10: Controller JSON regression

**Files:**
- Modify: `audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/library/web/AudiobookV1ControllerTest.java`

- [ ] **Step 1:** Add test `listReturnsBookAndCycleIds` returning DTO with `bookId(7L).cycleId(3L)` and assert `jsonPath("$[0].bookId").value(7)` and `jsonPath("$[0].cycleId").value(3)`.

- [ ] **Step 2:** Run `.\mvnw.cmd test -Dtest=AudiobookV1ControllerTest -q`.

---

## Plan self-review

| Spec section | Tasks covering |
|--------------|----------------|
| 4.1 Catalog bridge | 1, 2, 3, 10 |
| 4.2 `book_chapter` API | 4 |
| 4.3 Distribution dual-write | 5, 6, 7 |
| 4.4 `job_queue` audit | 8 |
| 4.5 Hygiene | 9 |
| Testing | embedded in 4, 5, 10 |

**Placeholder scan:** None intentional.

**Type consistency:** `bookId` / `cycleId` are `Long` in Java and `number` in TS JSON.

---

**Plan complete** — saved to `docs/superpowers/plans/2026-05-07-application-db-alignment.md`.

**Execution options:**

1. **Subagent-driven (recommended)** — one subagent per task (or per task group), review between tasks.  
2. **Inline** — run tasks sequentially in this chat with checkpoints after Tasks 4, 6, and 9.

Which approach do you want for implementation?
