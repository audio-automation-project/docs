# Plan A execution spine — package map (living document)

Populate during restructure (**before** bulk moves). Matches design spec **`§6` steps 2–4** (**CP-Arch** = step **3**).

**Inventory snapshot:** `2026-05-09-packages-before-restructure.txt`, `2026-05-09-inventory-files.tsv` (same folder). **Last spine scan:** 2026-05-09 (`audio-library-automation-bot` tree).

**Lockstep semantics**

- **`Spine lockstep group`:** Stable id for rows that share one **commit / merge boundary** (same contiguous sequence before anyone lands unrelated refactors).
- **`Must move together`:** **`Y`** = this row MUST ship in the **same contiguous commit series** as every other row with the **same group id**. **`Y (with …)`** = exact co-move set. **`N`** = can follow immediately after peer groups if imports allow, but list default pairing anyway.
- **PA-2 / `G1-MQ`:** **`@RabbitListener`**, serialized **`dto.message`** payloads, **publishers** (`ProcessingJobService` and any code enqueueing the same routes), and **`config.messaging`** (beans + **exchange/queue topology** + **routing/name constants**) are **one lockstep surface** during spine migration—do not split across PRs in a way that leaves half the stack on old packages or mismatched queue names.

## Primary table

| Spine role | Class | Spine lockstep group | Must move together | Filesystem path | Current package | Target package | Status |
|------------|-------|---------------------|---------------------|-----------------|---------------|----------------|--------|
| Rabbit ingress (listener) | `PipelineJobsRabbitListener` | **G1-MQ** | **Y** — all **G1-MQ** | `src/main/java/kg/automation/rest/automatation/infra/rabbit/PipelineJobsRabbitListener.java` | `kg.automation.rest.automatation.infra.rabbit` | `kg.automation.rest.automatation.service.job.listener` | landed |
| Rabbit wire payload | `PipelineJobMessage` | **G1-MQ** | **Y** — **G1-MQ** | `src/main/java/kg/automation/rest/automatation/infra/rabbit/PipelineJobMessage.java` | `kg.automation.rest.automatation.infra.rabbit` | `kg.automation.rest.automatation.dto.message.job` | landed |
| AMQP infra config | `InfraRabbitConfiguration` | **G1-MQ** | **Y** — **G1-MQ** | `src/main/java/kg/automation/rest/automatation/infra/rabbit/InfraRabbitConfiguration.java` | `kg.automation.rest.automatation.infra.rabbit` | `kg.automation.rest.automatation.config.messaging` | landed |
| Routing / queue constants | `PipelineRoutingKeys` | **G1-MQ** | **Y** — **G1-MQ** | `src/main/java/kg/automation/rest/automatation/infra/rabbit/PipelineRoutingKeys.java` | `kg.automation.rest.automatation.infra.rabbit` | `kg.automation.rest.automatation.config.messaging` | landed |
| Listener conditional | `PipelineJobsRabbitListenerEnabledCondition` | **G1-MQ** | **Y** — **G1-MQ** | `src/main/java/kg/automation/rest/automatation/infra/rabbit/PipelineJobsRabbitListenerEnabledCondition.java` | `kg.automation.rest.automatation.infra.rabbit` | `kg.automation.rest.automatation.config.messaging` | landed |
| Job dispatch / publish | `ProcessingJobService` | **G1-MQ** | **Y** — **G1-MQ** | `src/main/java/kg/automation/rest/automatation/jobs/service/ProcessingJobService.java` | `kg.automation.rest.automatation.jobs.service` | `kg.automation.rest.automatation.service.job` | landed |
| Pipeline message audit (publish path) | `PipelineMessageAuditBridge`, `PipelineMessageAuditService` | **G1-MQ** | **Y** — **G1-MQ** with `ProcessingJobService` | `src/main/java/.../infra/audit/PipelineMessageAuditBridge.java`, `.../PipelineMessageAuditService.java` | `kg.automation.rest.automatation.infra.audit` | `kg.automation.rest.automatation.service.job` (or split `service.job.audit` if preferred) | landed |
| Job executor | `PipelineJobExecutor` | **G2-EXEC** | **Y** — **G2-EXEC** | `src/main/java/kg/automation/rest/automatation/jobs/runtime/PipelineJobExecutor.java` | `kg.automation.rest.automatation.jobs.runtime` | `kg.automation.rest.automatation.service.job.runtime` | landed |
| Executor thread pool config | `PipelineJobExecutorConfig` | **G2-EXEC** | **Y** — **G2-EXEC** | `src/main/java/kg/automation/rest/automatation/config/PipelineJobExecutorConfig.java` | `kg.automation.rest.automatation.config` | `kg.automation.rest.automatation.config` or `service.job.runtime` (keep near executor) | landed |
| Audio runners | `AudioNoiseRemoveJobRunner` | **G2-EXEC** | **Y** — **G2-EXEC** | `src/main/java/kg/automation/rest/automatation/jobs/service/AudioNoiseRemoveJobRunner.java` | `kg.automation.rest.automatation.jobs.service` | `kg.automation.rest.automatation.service.job` | landed |
| Audio runners | `AudioTrimJobRunner`, `AudioCompressJobRunner`, `AudioDurationJobRunner`, `ConcatenateProcessingJobRunner` | **G2-EXEC** | **Y** — **G2-EXEC** | `jobs/service/AudioTrimJobRunner.java`, `AudioCompressJobRunner.java`, `AudioDurationJobRunner.java`, `ConcatenateProcessingJobRunner.java` | `kg.automation.rest.automatation.jobs.service` | `kg.automation.rest.automatation.service.job` | landed |
| Cleanup | `StaleProcessingJobCleanup` | **G2-EXEC** | Prefer **Y** — **G2-EXEC** | `src/main/java/kg/automation/rest/automatation/jobs/runtime/StaleProcessingJobCleanup.java` | `kg.automation.rest.automatation.jobs.runtime` | `kg.automation.rest.automatation.service.job.runtime` | landed |
| Orchestration | `PipelineOrchestrator` | **G3-ORCH** | **Y** — **G3-ORCH** | `src/main/java/kg/automation/rest/automatation/pipeline/orchestrator/PipelineOrchestrator.java` | `kg.automation.rest.automatation.pipeline.orchestrator` | `kg.automation.rest.automatation.service.pipeline.orchestrator` | landed |
| Spring event bridge → orchestrator | `JobCompletedEventListener` | **G3-ORCH** | **Y** — **G3-ORCH** | `src/main/java/kg/automation/rest/automatation/jobs/listener/JobCompletedEventListener.java` | `kg.automation.rest.automatation.jobs.listener` | `kg.automation.rest.automatation.service.job.listener` | landed |
| Repo | `ProcessingJobRepository` | **G4-REPO** | **Y** — **G4-REPO** | `src/main/java/kg/automation/rest/automatation/jobs/repository/ProcessingJobRepository.java` | `kg.automation.rest.automatation.jobs.repository` | `kg.automation.rest.automatation.repository.job` | landed |
| Repo | `PipelineRunRepository` | **G4-REPO** | **Y** — **G4-REPO** | `src/main/java/kg/automation/rest/automatation/jobs/repository/PipelineRunRepository.java` | `kg.automation.rest.automatation.jobs.repository` | `kg.automation.rest.automatation.repository.job` | landed |
| Repo | `AudioPartRepository` | **G4-REPO** | **Y** — **G4-REPO** | `src/main/java/kg/automation/rest/automatation/pipeline/repository/AudioPartRepository.java` | `kg.automation.rest.automatation.pipeline.repository` | `kg.automation.rest.automatation.repository.pipeline` | landed |
| Repo | `AudioStageRepository` | **G4-REPO** | **Y** — **G4-REPO** | `src/main/java/kg/automation/rest/automatation/pipeline/repository/AudioStageRepository.java` | `kg.automation.rest.automatation.pipeline.repository` | `kg.automation.rest.automatation.repository.pipeline` | landed |
| Layout helper | `CycleFileLayout` | **G5-LAYOUT** | **N** (after **G2–G4** compile) or **Y** with **G3-ORCH** | `src/main/java/kg/automation/rest/automatation/pipeline/service/CycleFileLayout.java` | `kg.automation.rest.automatation.pipeline.service` | `kg.automation.rest.automatation.service.pipeline` | landed |
| Integration client | `SilenceServiceClient` | **G6-INT** | **Y** — **G6-INT** | `src/main/java/kg/automation/rest/automatation/gateway/SilenceServiceClient.java` | `kg.automation.rest.automatation.gateway` | `kg.automation.rest.automatation.service.integration` | landed |
| Integration client | `WhisperServiceClient` | **G6-INT** | **Y** — **G6-INT** | `src/main/java/kg/automation/rest/automatation/gateway/WhisperServiceClient.java` | `kg.automation.rest.automatation.gateway` | `kg.automation.rest.automatation.service.integration` | landed |

**Cross-group:** **G1-MQ** should land **before or with** the first **G2-EXEC** consumer move that still compiles; **G4-REPO** typically **precedes** **G2-EXEC** / **G3-ORCH** if entities move first (see **PA-2** steps).

**Also track** spine-scoped **`model.entity`** types (same **G4-REPO** boundary as their repositories): `ProcessingJobEntity`, `PipelineRunEntity`, `AudioPartEntity`, audio-stage entities, `CycleEntity` if included in spine slice.

## Cross-check

Listener name **`PipelineJobsRabbitListener`** matches [`2026-05-08-pipeline-orchestrator-rabbitmq.md`](../../specs/2026-05-08-pipeline-orchestrator-rabbitmq.md) Plan A naming; update this artifact if the spec renames the class.
