# Plan A execution spine — package map (living document)

Populate during restructure (**before** bulk moves). Matches design spec **`§6` steps 2–4** (**CP-Arch** = step **3**).

**Lockstep semantics**

- **`Spine lockstep group`:** Stable id for rows that share one **commit / merge boundary** (same contiguous sequence before anyone lands unrelated refactors).
- **`Must move together`:** **`Y`** = this row MUST ship in the **same contiguous commit series** as every other row with the **same group id**. **`Y (with …)`** = exact co-move set. **`N`** = can follow immediately after peer groups if imports allow, but list default pairing anyway.
- **PA-2 / `G1-MQ`:** **`@RabbitListener`**, serialized **`dto.message`** payloads, **publishers** (`ProcessingJobService` and any code enqueueing the same routes), and **`config.messaging`** (beans + **exchange/queue topology** + **routing/name constants**) are **one lockstep surface** during spine migration—do not split across PRs in a way that leaves half the stack on old packages or mismatched queue names.

## Primary table

| Spine role | Class | Spine lockstep group | Must move together | Filesystem path | Current package | Target package | Status |
|------------|-------|---------------------|---------------------|-----------------|---------------|----------------|--------|
| Rabbit ingress (listener) | `PipelineJobsRabbitListener` | **G1-MQ** | **Y** — with all **G1-MQ** (wire + publisher + infra config) | _TBD_ | _TBD_ | `service.job.listener` (or `.messaging`) | pending |
| Rabbit wire payload(s) | e.g. `PipelineJobMessage` | **G1-MQ** | **Y** — **G1-MQ** (`dto.message.job`) | _TBD_ | _TBD_ | `dto.message.job` | |
| AMQP infra config | e.g. `InfraRabbitConfiguration`→`PipelineMessagingConfiguration` | **G1-MQ** | **Y** — **G1-MQ** | _TBD_ | _TBD_ | `config.messaging` | |
| Job dispatch / publish | `ProcessingJobService` | **G1-MQ** | **Y** — **G1-MQ** (publish path + audits touching same Rabbit constants) | | `...jobs.service` | `service.job` | |
| Job executor | `PipelineJobExecutor` | **G2-EXEC** | **Y** — with all **G2-EXEC** | `jobs/runtime/PipelineJobExecutor.java` | `...jobs.runtime` | `service.job.runtime` | |
| Audio runners | `AudioNoiseRemoveJobRunner` | **G2-EXEC** | **Y** — **G2-EXEC** | | `...jobs.service` | `service.job` | |
| Audio runners | `AudioTrimJobRunner`, `AudioCompressJobRunner`, `AudioDurationJobRunner`, `ConcatenateProcessingJobRunner` | **G2-EXEC** | **Y** — **G2-EXEC** | | | `service.job` | |
| Cleanup (optional same slice) | `StaleProcessingJobCleanup` | **G2-EXEC** | Prefer **Y** — **G2-EXEC** if wired to executor | | | `service.job.runtime` | |
| Orchestration | `PipelineOrchestrator` | **G3-ORCH** | **Y** — **G3-ORCH** with completion bridge | `pipeline/orchestrator/…` | `...pipeline.orchestrator` | `service.pipeline.orchestrator` | |
| Spring event bridge → orchestrator | `JobCompletedEventListener` | **G3-ORCH** | **Y** — **G3-ORCH** (same PR as orchestrator refactor) | | `...jobs.listener` | `service.job.listener` | |
| Repo | `ProcessingJobRepository` | **G4-REPO** | **Y** — **G4-REPO** (+ entities they map until stable) | | `...jobs.repository` | `repository.job` | |
| Repo | `PipelineRunRepository` | **G4-REPO** | **Y** — **G4-REPO** | _TBD_ | `...jobs.repository` | `repository.job` | |
| Repo | `AudioPartRepository` | **G4-REPO** | **Y** — **G4-REPO** | | `...pipeline.repository` | `repository.pipeline` | |
| Repo | `AudioStageRepository` | **G4-REPO** | **Y** — **G4-REPO** | _TBD_ | `...pipeline.repository` | `repository.pipeline` | |
| Layout helper | `CycleFileLayout` | **G5-LAYOUT** | **N** (after **G2–G4** compile) or **Y** with **G3-ORCH** if PR-churn small | `pipeline/service/CycleFileLayout.java` | `...pipeline.service` | `service.pipeline` | |
| Integration client | `SilenceServiceClient` | **G6-INT** | **Y** — both **G6-INT** clients | | `...gateway` | `service.integration` | |
| Integration client | `WhisperServiceClient` | **G6-INT** | **Y** — **G6-INT** | | | | |

**Cross-group:** **G1-MQ** should land **before or with** the first **G2-EXEC** consumer move that still compiles; **G4-REPO** typically **precedes** **G2-EXEC** / **G3-ORCH** if entities move first (see **PA-2** steps).

**Also track** spine-scoped **`model.entity`** types (same **G4-REPO** boundary as their repositories): `ProcessingJobEntity`, `PipelineRunEntity`, `AudioPartEntity`, audio-stage entities, `CycleEntity` if included in spine slice.
