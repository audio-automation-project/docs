# Spring Boot layered restructure — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move all `audio-library-automation-bot` Java code into the layered package layout from [`2026-05-09-springboot-layered-restructure-design.md`](../specs/2026-05-09-springboot-layered-restructure-design.md), fix package/path mismatches, rename typo’d public classes, delete obsolete duplicates, and lock rules with ArchUnit so the structure does not regress.

**Architecture:** One big-bang branch: migrate **model** → **repository** → **service** → **controller** + **dto**, then delete empty legacy trees. **Job instances** follow the same **one primary owner FK per job type** idea as in pipeline/schema docs (audio spine: `audio_part_id`; other families: see ER). **DTO layout** (parent spec §4.1): **`dto.request` / `dto.response`** for HTTP only; **`dto.message`** (`job`, `pipeline`, …) for queue and orchestration wire types; **`dto.event`** for event-shaped payloads when distinct; **`dto.internal`** for in-process non-messaging POCOs. **Normative mechanism placements** (parent spec §4.2): **`service.integration`** for outbound HTTP/RPC (`SilenceServiceClient`, `WhisperServiceClient`, other `gateway` clients); **`service.job`** (`runtime`, `listener`, `messaging` as needed) for `PipelineJobExecutor`, `*JobRunner`, stale-job cleanup, and **`@RabbitListener` dispatch**; **`service.pipeline.orchestrator`** for `PipelineOrchestrator` (business orchestration, **not** `integration`); **`config.messaging`** for Rabbit/AMQP **beans and topology only** (exchanges, queues, bindings, template wiring, infra constants)—so Plan A’s Rabbit path is not an orphan **`infra.rabbit`** package. FFmpeg and similar local processing adapters: **`service.integration`**. **Entities are not REST contracts** (§4.1): map to **`dto.request`/`dto.response`** at controller boundaries. **Existing cluster-boundary checks** in [`ClusterArchitectureTest`](../../../audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/architecture/ClusterArchitectureTest.java) must be updated to match new package segments. Follow parent spec **`§6` spine map + spine-first migration** (`PA-1` / `PA-2`) **before** broad layer sweeps. **Package drift prevention:** **`§10` ArchUnit** rules are mandatory CI gates—not advisory. **Checkpoint CP-Arch:** **`Task A1`** (**L1+L2+L4+LM**, **no `L3`**) merges **before** **or in the same PR/stack as `PA-2`** (see below). **`Task A2`** adds **`L3`** after **`model.entity`** migration completes.

**Tech Stack:** Java 17, Spring Boot 3.x, Maven, JUnit 5, ArchUnit 1.3.x (already in `pom.xml`).

**Parent spec:** `docs/superpowers/specs/2026-05-09-springboot-layered-restructure-design.md`

---

## File map (target packages)

All paths are under `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/` unless noted.

| Current area (illustrative) | Target layer / package |
|----------------------------|-------------------------|
| `pipeline.domain.*` (JPA `*Entity`) | `model.entity.pipeline` |
| `pipeline.domain.*` (enums, e.g. `UploadPlatform`, `FileLifecycle`) | `model.domain.pipeline` |
| `jobs.domain.*` (`*Entity` vs enum) — **`JobType` / `JobStatus` / `PipelineRunStatus`** are **job-slice** types (not `pipeline.domain`) | `model.entity.job` / `model.domain.job` |
| `library.domain.*` | `model.entity.library` |
| `pipeline.repository.*`, `jobs.repository.*`, `library.repository.*` | `repository.pipeline`, `repository.job`, `repository.library` |
| `pipeline.service.*`, `library.service.*`, `jobs.service.*`, … | `service.pipeline`, `service.library`, `service.job`, … |
| `pipeline.web.*`, `jobs.web.*`, `torrent.controller.*`, etc. | `controller.pipeline`, `controller.job`, `controller.torrent`, … |
| `pipeline.dto.*`, `library.dto.*` (HTTP) | `dto.request.*` / `dto.response.*` |
| Queue job payloads (`PipelineJobMessage`, etc.), orchestration wire POJOs | **`dto.message.job`**, **`dto.message.pipeline`** |
| Serialized event notifications (if distinct from commands) | **`dto.event.*`** |
| In-process helpers, non-wire POJOs | **`dto.internal.*`** |
| `gateway.*` (`SilenceServiceClient`, `WhisperServiceClient`, REST download helpers, etc.) | `service.integration` |
| `ffmpeg` (`FFmpegOperations`) | `service.integration` (local adapter, not pipeline domain orchestration) |
| `infra.rabbit.*` and any `infra.*` types used **only** for AMQP wiring or publish audit | **`config.messaging`**: connection, factories, declarables, exchange/queue constants (e.g. migrate `InfraRabbitConfiguration` into this package, optionally rename class); **`@RabbitListener`** classes → **`service.job.listener`** or **`service.job.messaging`**; job services that **publish** (`ProcessingJobService`) stay **`service.job`** and import from `config.messaging` |
| `jobs.runtime`, `jobs.listener`, `jobs.event` | `service.job.runtime`, `service.job.listener`, `model.domain.job` (**wire payloads**: `dto.message.job` per spec §4.1–§4.2) |
| `pipeline.orchestrator` (`PipelineOrchestrator`) | **`service.pipeline.orchestrator`** (required placement; not `integration`, not `config`) |
| `pipeline.scheduler` | `service.pipeline` |
| `ai.provider`, `ai.openai.controller`, `ai.text.controller`, `ai.image.service` | `service.ai`, `controller.ai`, `dto.request.ai` / `dto.response.ai` for HTTP |
| `image` DTO package (`BookCover*`) | `dto.response.ai` if returned on REST; else `dto.internal.ai` |
| `poster.youtube.service` (`YouTubeUploader`) | `service.distribution` |
| `telegram.service` | `service.distribution` |
| `search/baza_knig/controller` (`BookParserController`) | **`controller.pipeline` only** (frozen for this big bang—covers acquisition scraping as pipeline-adjacent surface; no **`controller.search`** slice). |
| `video.service`, `audio.service` | `service.pipeline` |
| `firebase.*` | `service.library` (audiobook mirror) and/or `service.integration` (generic Firestore job logging) — split by responsibility |
| `config.*` | `config` (unchanged root) |
| Root `AutomatationApplication` | `kg.automation.rest.automatation` (unchanged) |

**Tests:** Mirror the same package layout under `src/test/java/...`.

---

## Daily verification checklist (run every work session before push)

- [ ] `./mvnw -q -DskipTests compile` — exit code 0  
- [ ] `./mvnw -q test` — exit code 0  
- [ ] If touching HTTP layer: smoke `GET`/`POST` for cycle + library endpoints (document exact curls in your PR)  
- [ ] No Java file path disagrees with its `package` line (spot-check with ripgrep below)

**Path/package sanity (PowerShell, repo `audio-library-automation-bot`):**

```powershell
Get-ChildItem -Recurse -Filter *.java src\main\java,src\test\java | ForEach-Object {
  $pkg = (Select-String -Path $_.FullName -Pattern '^package\s+([\w.]+);' | Select-Object -First 1).Matches.Groups[1].Value
  if (-not $pkg) { return }
  $rel = $_.FullName -replace '.*\\java\\', '' -replace '\\', '.' -replace '\.java$', ''
  if ($rel -ne $pkg) { Write-Warning "MISMATCH: $($_.FullName) => package $pkg" }
}
```

Expected: no warnings.

---

## Rollback and stabilization

- [ ] **Step:** Before first move, tag: `git tag pre-layered-restructure-2026-05-09` (or branch `backup/pre-restructure`)  
- [ ] **Step:** If CI red and timeboxed: `git reset --hard` to tag or merge backup branch  
- [ ] **Step:** After green `mvn test`, merge via one PR; enable branch protection requiring green tests for 1–2 weeks post-merge  

---

## Inventory (complete list before coding)

**Files:**

- Create: `audio-library-automation-bot/scripts/restructure-inventory.ps1` (optional; or use Excel/Notion — must exist as deliverable artifact in repo or PR description)  
- Modify: N/A for pure inventory step  

- [ ] **Step 1: Export all packages**

Run from workspace root `audio-library-system` (so paths resolve correctly):

```powershell
cd e:\Projects\audio-library-system\audio-library-automation-bot
New-Item -ItemType Directory -Force -Path ..\docs\superpowers\plans\artifacts | Out-Null
Get-ChildItem -Recurse -Filter *.java src\main\java,src\test\java | ForEach-Object {
  $line = Select-String -Path $_.FullName -Pattern '^package ' | Select-Object -First 1
  if ($line) { $line.Line }
} | Sort-Object -Unique | Set-Content ..\docs\superpowers\plans\artifacts\2026-05-09-packages-before-restructure.txt
```

**Expected:** File lists one `package ...;` per line covering all compilation units.

- [ ] **Step 2: Build spreadsheet / table**

For each `.java` file, columns: `file path`, `current package`, `new package`, `action` (`MOVE` | `MOVE+RENAME` | `DELETE` | `KEEP`), `notes`.

**Expected:** Every one of ~174 main + ~47 test sources has a row.

- [ ] **Step 3: Commit**

```bash
cd e:/Projects/audio-library-system
git add docs/superpowers/plans/artifacts/2026-05-09-packages-before-restructure.txt
git commit -m "docs: snapshot package list before layered restructure"
```

(Adjust path if `docs` is versioned in a different repo.)

---

## Task PA-1: Plan A spine — package map (required before migration)

**Deliverable:** [`artifacts/2026-05-09-plan-a-spine-package-map.md`](artifacts/2026-05-09-plan-a-spine-package-map.md) (living table). Parent spec **`§6` step 2**.

- [ ] **Step 1: Fill mandatory rows (+ lockstep columns)**

For each type in the artifact mandatory set, populate **filesystem path**, **current package**, **target package**, plus:

- **`Spine lockstep group`** — **`G1-MQ`** (**`@RabbitListener`** ingress + **`dto.message.*`** wire payloads + **publish path** (`ProcessingJobService`, other publishers) + **`config.messaging`** beans/**topology**/**routing constants**—**all one lockstep** for PA-2), **`G2-EXEC`** (executor + audio runners (+ optional **`StaleProcessingJobCleanup`**)), **`G3-ORCH`** (**`PipelineOrchestrator`** + **`JobCompletedEventListener`**), **`G4-REPO`** + spine entities, **`G5-LAYOUT`**, **`G6-INT`** (gateway clients). Use artifact template verbatim.

- **`Must move together`** — **`Y (group …)`** for listener/wire/class/**`config`/publisher** cohesion; **`N`** where noted (e.g. **`CycleFileLayout`** timing per artifact).

- [ ] **Step 2: Cross-check Plan A rabbit spec**

Ensure listener name aligns with [`2026-05-08-pipeline-orchestrator-rabbitmq.md`](../specs/2026-05-08-pipeline-orchestrator-rabbitmq.md) and Phase A/B implementation plans—update artifact when class names diverge.

- [ ] **Step 3: Commit**

Commit the filled artifact wherever `docs/` is versioned.

---

## Checkpoint CP-Arch (**`Task A1` + `PA-2` sequencing**)

**Normative gate (mirrors parent spec **`§6` step 3`):**

- **`Task A1`** must introduce **`LayeredDependencyRulesTest`** with **at minimum `L1` + `L2`** (controller↔repository isolation). **Same A1 milestone also adds `L4`** (controllers do not depend on **`..model.entity..`**) and **`LM`** (`@RabbitListener` methods not declared under **`..config.messaging..`**). **`L3`** (entity residency) is **explicitly excluded** from A1 — it belongs to **`Task A2`** after **`model.entity`** migration.

- **Merged to the lineage that receives `PA-2` either** strictly **before** the first **`PA-2`** commit lands **`main`/`develop`**, **or** in **the identical PR/stack** as **`PA-2` Step 1**.

- **Forbidden:** **`PA-2` merged alone** with **zero** A1 enforcement.

Implementers may land **A1** in a prerequisite PR, then **`PA-2`**, **or** one stacked PR (**A1 + `PA-2`**)—CI must run **`LayeredDependencyRulesTest`** with **L1+L2+L4+LM** throughout **`PA-2`**.

---

## Task PA-2: Migrate Plan A spine slice first (**protected cohort**)

**Prerequisite:** **Checkpoint CP-Arch** satisfied (**`Task A1`** merged before/with this task).

**Rule:** Finish this vertical slice (**green tests**) **before** broad Task B+/C reorganisation of unrelated areas. Parent spec **`§6` step **4**.

**PA-2 Rabbit lockstep:** Move **`@RabbitListener`** classes, **`dto.message` job/payload types**, code that **publishes** to the job queue (**`ProcessingJobService`** and related), and **`config.messaging`** (Spring AMQP beans, **declared exchanges/queues/bindings**, **routing-key and name constants**) in the **same contiguous commit series** (spine artifact **G1-MQ**). Do not merge a half-updated Rabbit vertical—reviewers should see listener, payloads, publisher, and topology/constants cross the boundary together.

- [ ] **Step 1: Move spine-scoped `model.entity` + `dto.message` (+ `config.messaging` classes)**

Entities referenced **only through** spine repos/runners—or shared kernel (`CycleEntity`, etc.) strictly required—move per **PA-1** artifact target column. Migrate **`infra.rabbit`→`config.messaging`** if touched by **`ProcessingJobService`**.

- [ ] **Step 2: Move spine repositories**

`ProcessingJobRepository`, `PipelineRunRepository`, `AudioPartRepository`, `AudioStageRepository` → `repository.job` / `repository.pipeline`; fix imports inside spine services.

- [ ] **Step 3: Move spine services**

`PipelineJobExecutor`, audio `*Runner`, `ProcessingJobService`, `PipelineOrchestrator`, `CycleFileLayout`, gateways → `service.job`, `service.pipeline.orchestrator`, `service.pipeline`, `service.integration`; listener → `service.job.listener`. Wire **`PipelineJobsRabbitListener`** when landed.

- [ ] **Step 4: Spine regression tests**

```bash
cd audio-library-automation-bot
./mvnw -q test -Dtest=PipelineOrchestratorPlanBTest,PipelineOrchestratorAudioTest,ProcessingJobServiceTest,\
AudioNoiseRemoveJobRunnerTest,AudioTrimJobRunnerTest,AudioCompressJobRunnerTest,\
AudioDurationJobRunnerTest,ConcatenateJobPathDerivationTest,CycleFileLayoutTest
```

**Expected:** all selected tests **PASS**. Add `-Dtest=...InternalPipelineJobControllerTest` if internal job API overlaps spine.

- [ ] **Step 5: Commit**

```bash
git commit -am "refactor: migrate Plan A spine slice (orchestrator, jobs, audio repos, integration clients)"
```

---

## Task A1: ArchUnit — introduce **`LayeredDependencyRulesTest`** (**L1, L2, L4, LM** — **no L3**)

**Parent:** spec **`§10`**; merges under **Checkpoint CP-Arch**.

Use **`archunit-junit5`**. **`L1`/`L2`:** layered isolation. **`L4`:** controllers allowed **`dto.request`**, **`dto.response`**, **`service`**, **`model.domain`**—**must not depend on **`..model.entity..`** packages** (proxy for banning **`@Entity` in public signatures`). **`L3`** (**`Task A2`**) completes entity-package residency checks. **`Task J`** (controller ripgrep) plus **`L4`/`L3`** together enforce REST entity-leak prevention—**CI ArchUnit and `Task J` are both required**, not alternatives. **`LM`:** **`config.messaging`** holds beans/constants/topology only — **no** `@RabbitListener` **declarations** there.

**Files:**

- Create or replace: `audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/architecture/LayeredDependencyRulesTest.java`  
- Optional later: `Architectures.layeredArchitecture()` freeze.  
- Modify: [`ClusterArchitectureTest`](../../../audio-library-automation-bot/src/test/java/kg/automation/rest/automatation/architecture/ClusterArchitectureTest.java) (**Task H**).  

- [ ] **Step 1: Create test class containing L1 + L2 + L4 + LM (omit L3)**

```java
package kg.automation.rest.automatation.architecture;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import kg.automation.rest.automatation.AutomatationApplication;
import org.junit.jupiter.api.DisplayName;
import org.springframework.amqp.rabbit.annotation.RabbitListener;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.methods;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@DisplayName("Layered architecture (ArchUnit) — Task A1 (L1, L2, L4, LM)")
@AnalyzeClasses(packagesOf = AutomatationApplication.class, importOptions = ImportOption.DoNotIncludeTests.class)
class LayeredDependencyRulesTest {

    /** L1 — controllers never touch repositories directly. */
    @ArchTest
    static final ArchRule controllersMustNotDependOnRepositories =
            noClasses()
                    .that().resideInAnyPackage("..controller..")
                    .should().dependOnClassesThat().resideInAnyPackage("..repository..");

    /** L2 — repositories never reference HTTP controllers. */
    @ArchTest
    static final ArchRule repositoriesMustNotDependOnControllers =
            noClasses()
                    .that().resideInAnyPackage("..repository..")
                    .should().dependOnClassesThat().resideInAnyPackage("..controller..");

    /**
     * L4 — controllers must not pull in persistence entity packages (REST surface uses dto + services + domain;
     * no model.entity exposure). If a justified exception appears, narrow with {@code because} + ticket reference.
     */
    @ArchTest
    static final ArchRule controllersMustNotDependOnEntityPackages =
            noClasses()
                    .that().resideInAnyPackage("..controller..")
                    .should().dependOnClassesThat().resideInAnyPackage("..model.entity..");

    /**
     * LM — Rabbit listeners belong in {@code service.job} (or allied packages), never under {@code config.messaging}.
     * Adjust predicate if ArchUnit/API differs ({@code areAnnotatedWith} vs meta-annotation).
     */
    @ArchTest
    static final ArchRule rabbitListenerMethodsMustNotBeDeclaredInConfigMessaging =
            methods()
                    .that().areAnnotatedWith(RabbitListener.class)
                    .should()
                    .beDeclaredInClassesThat()
                    .resideOutsideOfPackages("..config.messaging..");
}
```

Until **`..config.messaging..`** packages exist post-move, **`LM`** may report **vacuous PASS** (`allowEmptyShould` not needed when methods rule matches zero `@RabbitListener` methods—but if ArchUnit treats empty differently, consult ArchUnit docs).

- [ ] **Step 2: Run tests**

```bash
cd audio-library-automation-bot
./mvnw -q test -Dtest=LayeredDependencyRulesTest
```

**Expected:** COMPILATION **GREEN**. If **`beDeclaredInClassesThat().resideOutsideOfPackages`** chain mismatches ArchUnit **`1.3.x`** API, refactor to **`methods().that()...should().notBeDeclaredInClassesThat(resideInAPackage(..config.messaging..))`** equivalents.

**Expected tests:** GREEN after violations fixed.

- [ ] **Step 3: Commit (A1 / CP-Arch bundle)**

```bash
git commit -am "test: ArchUnit A1 rules L1 L2 L4 LM (no L3) per spec §10"
```

---

## Task A2: ArchUnit — add **`L3`** after **`model.entity`** migration (**post Task B**) 

**Timing:** Execute after **every** JPA **`@Entity`** has been relocated under **`kg.automation.rest.automatation.model.entity..`** (including **`PA-2`** spine entities + **`Task B`** remainder). **Prior A1:** keep **`L1` + `L2` + `L4` + `LM`** enforced.

- [ ] **Step 1: Append **`L3`** to `LayeredDependencyRulesTest.java`**

```java
import jakarta.persistence.Entity;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;

    /** L3 — JPA entities only under {@code ..model.entity..}. Task A2 only. */
    @ArchTest
    static final ArchRule entitiesMustResideInModelEntity =
            classes().that().areAnnotatedWith(Entity.class).should().resideInAnyPackage("..model.entity..");
```

- [ ] **Step 2: Run tests**

```bash
cd audio-library-automation-bot
./mvnw -q test -Dtest=LayeredDependencyRulesTest
```

- [ ] **Step 3: Commit**

```bash
git commit -am "test: ArchUnit A2 add L3 entity residency per spec §10"
```

---

## Task B: Model layer (**remainder** — pipeline + library + job + distribution entities not migrated in PA-2)

**Convention:** `@Entity` classes → `model.entity.<feature>`; other stable domain types → `model.domain.<feature>`.

**Overlap with PA-2:** Do **not** double-move spine entities (**`ProcessingJobEntity`**, **`PipelineRunEntity`**, **`AudioPartEntity`**, audio-stage–related mappings, **`CycleEntity`** if PA-2 took them)—complete only leftovers here.

**Files (execute moves; adjust if your inventory differs):**

- Move/rename packages for all classes currently in:
  - `..pipeline.domain..` → split entity vs domain
  - `..library.domain..` → `model.entity.library`
  - `..jobs.domain..` → `@Entity` → `model.entity.job`; **`JobType` / `JobStatus` / `PipelineRunStatus`** → **`model.domain.job`**
- Move misplaced “entity” types that are under `audio.dto`, `video.dto` but declare `pipeline.domain`:
  - `audio/dto/AudioPartEntity.java` → `model/entity/pipeline/AudioPartEntity.java`
  - `video/dto/VideoStageEntity.java`, `VideoPartEntity.java` → `model/entity/pipeline/`
- Fix `ai/image/dto/ImagePartType.java` — file lives under `ai/image/dto` but package is `pipeline.domain` → move to `model/domain/pipeline/ImagePartType.java` with matching package.

- [ ] **Step 1: Move `LibraryAudiobookEntity`**

From: `library/domain/LibraryAudiobookEntity.java`  
To: `model/entity/library/LibraryAudiobookEntity.java`  
Change: `package kg.automation.rest.automatation.model.entity.library;`  
Update every `import` across `src/main/java` and `src/test/java` (IDE refactor “Move” preferred).

- [ ] **Step 2: Split `pipeline.domain` bundle**

Example mappings:

| Class | New package |
|-------|-------------|
| `CycleEntity`, `OverlayAssetEntity`, `BookChapterEntity`, `DistributionEntity`, `DistributionResultEntity`, `UploadResultEntity`, `AudioPartEntity`, `VideoPartEntity`, `VideoStageEntity` | `model.entity.pipeline` |
| `JobType`, `JobStatus`, `PipelineRunStatus` | **`model.domain.job`** (canonical; not `dto.*`) |
| `UploadPlatform`, `FileLifecycle`, other pipeline enums/value objects | `model.domain.pipeline` |

- [ ] **Step 3: Compile**

```bash
./mvnw -q -DskipTests compile
```

**Expected:** BUILD SUCCESS (imports may need iteration until all references updated).

- [ ] **Step 4: Commit**

```bash
git commit -am "refactor: move JPA entities to model.entity.* packages"
```

---

## Task C: Repository layer

**Overlap:** Spine repos (**`ProcessingJobRepository`**, **`PipelineRunRepository`**, **`AudioPartRepository`**, **`AudioStageRepository`**) should already match targets after **`PA-2`**—skip if done.

- [ ] **Step 1: Move interfaces**

`pipeline.repository.*` → `repository.pipeline`  
`jobs.repository.*` → `repository.job`  
`library.repository.*` → `repository.library`

Update `import` statements in all services and tests.

- [ ] **Step 2: Compile and test**

```bash
./mvnw -q test
```

**Expected:** BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git commit -am "refactor: move Spring Data repositories to repository.*"
```

---

## Task D: Service layer — pipeline, job, library, ai, integration, distribution

**Files (high level):**

- `pipeline.service.*` → `service.pipeline` (retain subpackages like `provider.distribution` → `service.distribution.provider` **or** flatten to `service.distribution` — prefer **flatten** for spec alignment: `PlatformUploader` in `service.distribution`)
- `jobs.service.*`, `jobs.runtime.*`, `jobs.listener.*`, `jobs.event.*` → `service.job` (use subpackages `service.job.runtime`, `service.job.listener`, `service.job.event` if needed)
- `library.service.*` → `service.library`
- `video.service.*`, `audio.service.*` → `service.pipeline`
- `pipeline.orchestrator.*` → **`service.pipeline.orchestrator`** (same for `PipelineOrchestrator.java`; do not merge into `integration`)
- `pipeline.scheduler.*` → `service.pipeline.scheduler`
- `infra.rabbit.*` → **`config.messaging`** (beans/topology/constants only)
- `gateway.*` → `service.integration`
- `ffmpeg` → `service.integration` (package `kg.automation.rest.automatation.service.integration`)
- `ai.provider`, `ai.image.service` → `service.ai` and `service.ai.image` (optional subpackage)
- `image.service.BookCoverService` → `service.ai` (cover generation is AI-facing)
- `poster.youtube.service` → `service.distribution`
- `telegram.service` → `service.distribution`

- [ ] **Step 1: Move `DistributionPersistenceService`**

To: `service/distribution/DistributionPersistenceService.java`  
Package: `kg.automation.rest.automatation.service.distribution`

- [ ] **Step 2: Move `PipelineJobExecutor` and all `*JobRunner` / `ProcessingJobService` / `StaleProcessingJobCleanup`**

To: `service/job/runtime/`, `service/job/`, etc. per parent spec §4.2. Ensure **no** `@RabbitListener` remains under `config.messaging`—only under `service.job.listener` (or `service.job.messaging`).

- [ ] **Step 3: Move `PipelineOrchestrator`**

To: `service/pipeline/orchestrator/PipelineOrchestrator.java`  
Package: `kg.automation.rest.automatation.service.pipeline.orchestrator`

- [ ] **Step 4: Collapse `infra.rabbit` into `config.messaging`**

Move each class from `kg.automation.rest.automatation.infra.rabbit` to `kg.automation.rest.automatation.config.messaging` (adjust `import` in `ProcessingJobService`, tests, and any listener). Delete the empty `infra.rabbit` package when done. Rename `InfraRabbitConfiguration` → `PipelineMessagingConfiguration` (or keep name if you prefer minimal diff—**package** must be `config.messaging`).

- [ ] **Step 5: Full compile**

```bash
./mvnw -q -DskipTests compile
```

- [ ] **Step 6: Full test**

```bash
./mvnw -q test
```

- [ ] **Step 7: Commit**

```bash
git commit -am "refactor: consolidate services under service.* and config.messaging"
```

---

## Task E: DTO layer

- [ ] **Step 1: Move pipeline DTOs**

`pipeline/dto/CycleDto.java` → `dto/response/pipeline/CycleDto.java` with package `kg.automation.rest.automatation.dto.response.pipeline`

`BookChapterUpsertRequest` → `dto/request/pipeline`  
`BookChapterResponse` → `dto/response/pipeline`  
`ScrapedBookResponse` → `dto/response/pipeline`

- [ ] **Step 2: Move library DTOs**

`library/dto/AudiobookDto.java` → `dto/response/library`

- [ ] **Step 3: Move image/OpenAI DTOs**

`ai/image/dto/BookCover*.java` → `dto/response/ai` (or `dto/internal/ai` if not exposed on REST)

- [ ] **Step 4: Torrent request types**

`torrent/controller/TorrentUploadRequest.java` → `dto/request/torrent/TorrentUploadRequest.java`

- [ ] **Step 5: Messaging / orchestration payloads (`dto.message`, optional `dto.event`)**

Place queue bodies and dispatched job payloads (e.g. `PipelineJobMessage`) under **`dto/message/job`** with package `kg.automation.rest.automatation.dto.message.job`. Place orchestration wire types consumed across async/runner boundaries under **`dto/message/pipeline`** (`...dto.message.pipeline`). Serialized **notification** payloads that are intentionally event-shaped → **`dto/event/job`** (or merge into `dto.message` if unused). Keep routing-key **constants/enums** in **`config.messaging`** when purely transport—not in `dto.message`.

- [ ] **Step 6: Test**

```bash
./mvnw -q test
```

- [ ] **Step 7: Commit**

```bash
git commit -am "refactor: split dto into request/response/message/event/internal"
```

---

## Task F: Controller layer + REST entrypoints

- [ ] **Step 0: REST boundary audit**

Ensure no controller exposes **`model.entity`** types (see spec §4.1). Replace signatures with **`dto.request`** / **`dto.response`**; keep **`dto.message`** for listener/orchestration code paths outside REST.

- [ ] **Step 1: Move controllers**

`pipeline/web/*` → `controller/pipeline`  
`jobs/web/*` → `controller/job`  
`torrent/controller/*` → `controller/torrent` (split: DTOs already in Task E)  
`ai/openai/controller/OpenAiCotroller.java` → `controller/ai/OpenAiController.java` (**rename class and file**)

`ai/text/controller/TextExtractorController.java` declares package `text.controller` — move file to `controller/ai/TextExtractorController.java`, package `kg.automation.rest.automatation.controller.ai`.

`search/baza_knig/controller/BookParserController.java` → `controller/pipeline/BookParserController.java` (frozen target — **`controller.pipeline`**).

- [ ] **Step 2: Fix Spring component scan**

`AutomatationApplication` is in root package — Spring Boot scans `kg.automation.rest.automatation` and subpackages by default. After moves, controllers must stay under that base. **Expected:** no `@SpringBootApplication` scan change.

- [ ] **Step 3: Run controller tests**

```bash
./mvnw -q test -Dtest=CycleLifecycleEndpointTest,InternalPipelineJobControllerTest,AudiobookV1ControllerTest,JobLogV1ControllerTest
```

**Expected:** All pass.

- [ ] **Step 4: Commit**

```bash
git commit -am "refactor: move REST controllers to controller.*; rename OpenAiController"
```

---

## Task G: Typo rename — text extractor service

- [ ] **Step 1: Locate class**

Search: `ImageDescriptionTextExtractorSercvice`  
Rename to: `ImageDescriptionTextExtractorService`  
Move to: `service/ai/` or `service/pipeline/` depending on ownership (text extraction from images is AI-assisted → **`service.ai`**).

- [ ] **Step 2: Update references**

```bash
rg "ImageDescriptionTextExtractorSercvice" audio-library-automation-bot
```

**Expected:** no matches.

- [ ] **Step 3: Test**

```bash
./mvnw -q test
```

- [ ] **Step 4: Commit**

```bash
git commit -am "refactor: rename ImageDescriptionTextExtractorService"
```

---

## Task H: Config, Firebase, tests, ClusterArchitectureTest

- [ ] **Step 1: Move Firebase services**

Under `firebase/service/` → `service.library`, `service.integration`, or `service.job` by responsibility (audiobook doc sync vs job log vs image log).

- [ ] **Step 2: Update `ClusterArchitectureTest` package predicates**

Replace rules that rely on obsolete segments (e.g. `..distribution..` only if packages still contain that token). Example: if torrent lives under `controller.torrent`, rule “distribution must not depend on torrent” should use `..service.distribution..` → `..service.torrent..` or `..controller.torrent..` as appropriate.

Concrete starting point after migration:

```java
noClasses()
    .that().resideInAnyPackage("..service.distribution..", "..controller.distribution..")
    .should().dependOnClassesThat().resideInAnyPackage("..controller.torrent..", "..service.torrent..")
    .allowEmptyShould(true);
```

Adjust after real package graph is stable.

- [ ] **Step 3: Move integration tests**

`src/test/java/.../integration/*.java` may stay package `..integration..` **or** move to `..integration.pipeline..` — optional; ArchUnit excludes tests via `ImportOption.DoNotIncludeTests`.

- [ ] **Step 4: Run full suite**

```bash
./mvnw -q test
```

- [ ] **Step 5: Commit**

```bash
git commit -am "refactor: align firebase and architecture tests with new packages"
```

---

## Task I: Aggressive deletion — empty and duplicate trees

**Files:**

- Delete: obsolete directories under `src/main/java/kg/automation/rest/automatation/` that have zero `.java` files after migration (e.g. empty `pipeline/web`, `jobs/web`, legacy `youtube/service`, `image`, `gateway`, `poster`, `ai/text`, `ai/openai`)

- [ ] **Step 1: Verify no imports point at deleted packages**

```bash
rg "kg\\.automation\\.rest\\.automatation\\.(pipeline\\.web|jobs\\.web|gateway|poster\\.youtube)"
 audio-library-automation-bot/src
```

**Expected:** no matches (or fix stragglers).

- [ ] **Step 2: Delete empty dirs** (Git does not track empty dirs — removing last file is enough)

- [ ] **Step 3: Commit**

```bash
git commit -am "chore: remove empty legacy packages after layered restructure"
```

---

## Task J: Final gates + smoke

**`Task J` Step 3** (controller **`model.entity`** ripgrep) **complements** **ArchUnit `L4`** (**`Task A1`**) and **`L3`** (**`Task A2`**) for **entity-leak prevention**—keep all three in the release checklist.

- [ ] **Step 1: Compile**

```bash
./mvnw -q -DskipTests compile
```

- [ ] **Step 2: Tests**

```bash
./mvnw -q test
```

- [ ] **Step 3: DTO / entity REST leak spot-check (concrete rg; pairs with ArchUnit `L4` + `L3`)**

Run from **`audio-library-automation-bot`** (expect **zero matching lines**; **ripgrep** typically exits **`1`** when there are zero matches—which is OK here):

```powershell
rg "public .*model\\.entity\\." src/main/java/kg/automation/rest/automatation/controller
rg "import .*\\.model\\.entity\\." src/main/java/kg/automation/rest/automatation/controller
```

**Purpose:** catch **`public`** methods that still mention **`model.entity`** in signatures (first pattern) and **direct entity imports** in controllers (second). Extend patterns if generics or fully-qualified names sneak past.

Legacy packages pre-migration (optional until fully moved):

```powershell
rg "import kg\\.automation\\.rest\\.automatation\\.(pipeline\\.domain|jobs\\.domain|library\\.domain)\\.[A-Za-z0-9_]+Entity\\b" src/main/java/kg/automation/rest/automatation/controller
```

**Expected:** no matches after rewrite (controllers rely on **`dto.request` / `dto.response`**).

Also confirm **`dto.message.*`** holds queue orchestration payloads—not misplaced under **`dto.request`/`dto.response`** (spot `rg "^package .*\\.dto\\.message"` vs request/response).

- [ ] **Step 4: Smoke (example — adjust ports/paths)**

With PostgreSQL up per runbook:

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments="--app.core.enabled=true"
```

In another terminal:

```bash
curl -sSf http://localhost:8088/actuator/health
curl -sSf http://localhost:8088/api/v1/library/audiobooks
```

(Second URL path must match actual `AudiobookV1Controller` mapping — **replace** if different.)

**Expected:** HTTP 200 or documented auth behavior.

- [ ] **Step 5: Update spec status**

Modify `docs/superpowers/specs/2026-05-09-springboot-layered-restructure-design.md` header: `Status: Implemented` and link merging PR.

- [ ] **Step 6: Commit**

```bash
git commit -am "docs: mark layered restructure spec implemented"
```

---

## Plan self-review

| Spec section | Covered by |
|--------------|------------|
| Target blueprint | File map + Tasks B–F |
| DTO taxonomy (§4.1): `request`/`response`/`message`/`event`/`internal` | Architecture preamble + File map rows + Task E Step 5 |
| Mechanism placements (§4.2): integration / job / pipeline orchestrator / `config.messaging` | Architecture preamble + File map rows + Task D Steps 2–4 |
| Plan A spine (**§6** steps **2–4** + **CP-Arch** step **3**) | **PA-1**, **Checkpoint CP-Arch**, **`Task A1`**, **`PA-2`** + artifact [`artifacts/2026-05-09-plan-a-spine-package-map.md`](artifacts/2026-05-09-plan-a-spine-package-map.md) |
| ArchUnit / package guards (**§10**) | **`Task A1`** (**L1, L2, L4, LM**) + **`Task A2`** (**`L3`**), spec §10 table |
| Entity vs API (§4.1) | Task F Step 0 + spec §7 gate + **ArchUnit `L4`/`L3`** (**A1/A2**) **+** Task J Step 3 (rg)—**combined** enforcement |
| Delete policy | Task I |
| **CP-Arch** | **Checkpoint CP-Arch**: **`Task A1` (`L1`+`L2`+`L4`+`LM`) BEFORE or MERGED WITH `PA-2`** (**no `L3` in A1**) |
| Big-bang sequence + spine-first slice | Inventory → **`PA-1`** → **`Checkpoint CP-Arch`** + **`Task A1`** → **`PA-2`** → **`Task B`–`F`** breadth; **`Task A2`** (`L3`) after **`model.entity`** migration completes; Tasks **G → J** |
| Verification gates | Daily checklist + Task J + **ArchUnit `§10`** |
| Risk / rollback | Rollback section + Tasks **A1** / **A2** rules |
| ArchUnit guard | Tasks **A1**, **A2**, **H** |

**Placeholder scan:** No TBD/TODO sections; **`BookParserController`** target **`controller.pipeline`** is frozen (**no `controller.search` slice for this refactor**).

---

**Plan complete and saved to `docs/superpowers/plans/2026-05-09-springboot-layered-restructure-implementation-plan.md`.**

**Two execution options:**

1. **Subagent-driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration (`subagent-driven-development` skill).

2. **Inline execution** — run tasks in this session using checkpoints (`executing-plans` skill).

Which approach do you want for implementation?
