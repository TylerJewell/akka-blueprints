# PLAN — sk-process

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab with the Akka theme variables and the Lesson 24 state-label CSS overrides.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Coord[ProcessCoordinator]:::agent
  Planner[StepPlanner]:::agent
  Exec[StepExecutor]:::agent
  Checker[QualityChecker]:::agent

  OrchWF[ProcessOrchestrationWorkflow]:::wf
  ExecWF[ExecutorWorkflow]:::wf

  Job[JobEntity]:::ese
  Step[StepEntity]:::ese
  Queue[JobQueue]:::ese
  JBoard[JobBoardView]:::view
  SBoard[StepBoardView]:::view
  JobC[JobRequestConsumer]:::cons
  EvalC[StepEvalConsumer]:::cons
  Sim[JobSimulator]:::ta
  Stuck[StuckStepMonitor]:::ta
  API[JobQueueEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit job| Queue
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| JobC
  JobC -->|create| Job
  JobC -->|start workflow| OrchWF

  OrchWF -->|PLAN_JOB / FINALISE_JOB| Coord
  OrchWF -->|DETAIL_PLAN| Planner
  OrchWF -->|CHECK_QUALITY x criteria| Checker
  OrchWF -->|create one per step| Step
  OrchWF -->|record stages| Job
  OrchWF -->|poll steps| SBoard
  OrchWF -->|pause / resume| Job

  ExecWF -->|poll board| SBoard
  ExecWF -->|atomic claim| Step
  ExecWF -->|EXECUTE_STEP| Exec

  Exec -.->|StepTools write| Step
  Checker -.->|StepTools write| Job

  Job -.->|projects| JBoard
  Step -.->|projects| SBoard
  Job -.->|stage events| EvalC
  EvalC -->|record signal| Job
  Stuck -.->|every 45s| Step

  API -->|query / SSE| JBoard
  API -->|query / SSE| SBoard
  API -->|pause / resume / audit-review| Job
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions, scheduled ticks, and guarded tool writes. `StepExecutor` and `QualityChecker` are each one agent class run as several instances — `executor-1`/`executor-2`/`executor-3` and one checker per criterion (`format`, `completeness`, `policy`). The top-level `ProcessOrchestrationWorkflow` is the coordinator; the three desks each run a different internal coordination capability: planning delegates to the planner agent, execution is a team over the shared `StepBoardView`, validation is a moderated panel feeding `QualityRule`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as JobQueueEndpoint
  participant Q as JobQueue
  participant C as JobRequestConsumer
  participant OW as ProcessOrchestrationWorkflow
  participant PC as ProcessCoordinator
  participant SP as StepPlanner
  participant S as StepEntity
  participant EW as ExecutorWorkflow
  participant EX as StepExecutor
  participant QC as QualityChecker
  participant J as JobEntity

  U->>API: POST /api/jobs {template}
  API->>Q: submitJob(request)
  API-->>U: 202 {jobId}
  Q->>C: JobSubmitted
  C->>J: createJob
  C->>OW: start({jobId})
  OW->>PC: PLAN_JOB(template)
  PC-->>OW: StepPlan{objective, steps}
  OW->>J: recordPlan
  OW->>SP: DETAIL_PLAN
  SP-->>OW: DetailedPlan{approach, steps}
  OW->>J: recordDetailedPlan
  Note over OW: writes one StepEntity per step (OPEN)
  OW->>S: createStep x N
  OW->>J: initialiseSteps
  Note over EW: executor loops already polling the board
  EW->>S: claim(executor-1) (atomic, single winner)
  EW->>EX: EXECUTE_STEP(step)
  EX-->>S: writeStepResult (guarded)
  EW->>S: recordDone
  Note over OW: poll until all steps DONE
  OW->>J: recordBatch
  OW->>QC: CHECK_QUALITY(criterion, batch) x 3
  QC-->>J: appendCheckNote (guarded)
  OW->>OW: QualityRule(notes) -> PASS
  OW->>J: recordVerdict(PASS)
  OW->>PC: FINALISE_JOB(batch)
  Note over PC: before-agent-response guardrail vets the JobSummary
  PC-->>OW: JobSummary
  OW->>J: finalise(summary, ref)
  J-->>API: SSE status=COMPLETED
```

## State machine — `JobEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED: createJob
  SUBMITTED --> PLANNED: recordPlan
  PLANNED --> EXECUTING: initialiseSteps
  EXECUTING --> PAUSED: pauseJob (operator signal)
  PAUSED --> EXECUTING: resumeJob (operator signal)
  EXECUTING --> VALIDATING: recordBatch (all steps done)
  VALIDATING --> APPROVED: recordVerdict PASS
  VALIDATING --> EXECUTING: requestRetry (one bounded round)
  APPROVED --> COMPLETED: finalise (output guardrail passes)
  APPROVED --> APPROVED: recordFinaliseBlock (guardrail refusal)
  COMPLETED --> COMPLETED: recordAuditReview (post-completion, non-blocking)
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  JobEntity ||--o{ JobCreated : emits
  JobEntity ||--o{ StepPlanned : emits
  JobEntity ||--o{ DetailedPlanRecorded : emits
  JobEntity ||--o{ StepsInitialised : emits
  JobEntity ||--o{ BatchCompleted : emits
  JobEntity ||--o{ ValidationCompleted : emits
  JobEntity ||--o{ RetryRequested : emits
  JobEntity ||--o{ JobFinalised : emits
  JobEntity ||--o{ JobPaused : emits
  JobEntity ||--o{ JobResumed : emits
  JobEntity ||--o{ StepSignalRecorded : emits
  JobEntity ||--o{ AuditReviewRecorded : emits
  JobBoardView }o--|| JobEntity : projects
  StepEvalConsumer }o--|| JobEntity : subscribes
  JobEntity ||--o{ StepEntity : "owns N steps"
  StepEntity ||--o{ StepCreated : emits
  StepEntity ||--o{ StepClaimed : emits
  StepEntity ||--o{ StepDone : emits
  StepEntity ||--o{ StepReleased : emits
  StepBoardView }o--|| StepEntity : projects
  JobQueue ||--o{ JobSubmitted : emits
  JobRequestConsumer }o--|| JobQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ProcessCoordinator` | `application/ProcessCoordinator.java` |
| `StepPlanner` | `application/StepPlanner.java` |
| `StepExecutor` | `application/StepExecutor.java` |
| `QualityChecker` | `application/QualityChecker.java` |
| `ProcessTasks` | `application/ProcessTasks.java` |
| `StepTools` | `application/StepTools.java` |
| `QualityRule` | `application/QualityRule.java` |
| `StepEvaluator` | `application/StepEvaluator.java` |
| `ProcessOrchestrationWorkflow` | `application/ProcessOrchestrationWorkflow.java` |
| `ExecutorWorkflow` | `application/ExecutorWorkflow.java` |
| `JobEntity` | `application/JobEntity.java` (state in `domain/Job.java`, events in `domain/JobEvent.java`) |
| `StepEntity` | `application/StepEntity.java` (state in `domain/Step.java`, events in `domain/StepEvent.java`) |
| `JobQueue` | `application/JobQueue.java` |
| `JobBoardView` | `application/JobBoardView.java` |
| `StepBoardView` | `application/StepBoardView.java` |
| `JobRequestConsumer` | `application/JobRequestConsumer.java` |
| `StepEvalConsumer` | `application/StepEvalConsumer.java` |
| `JobSimulator` | `application/JobSimulator.java` |
| `StuckStepMonitor` | `application/StuckStepMonitor.java` |
| `JobQueueEndpoint` | `api/JobQueueEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **4 autonomous-agent · 2 workflow · 3 event-sourced-entity · 2 view · 2 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Two coordination primitives sit under one pipeline.** The top-level `ProcessOrchestrationWorkflow` is sequential delegation: each stage runs and writes its result onto the shared `JobEntity` before the next begins. The execution stage hands off to an independent team: the workflow seeds the board and then waits, while the per-executor `ExecutorWorkflow` loops claim and fill steps on their own clock.
- **Atomic claim is the execution-team primitive.** `StepEntity` is a single-writer; `claim(executorId)` emits `StepClaimed` only when the current status is `OPEN`. Two executor workflows that read the same `OPEN` step from the board and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `Step` and returns to polling. No lock, no external queue.
- **The execution wait is a poll, not a block.** `ProcessOrchestrationWorkflow.executionStep` queries `StepBoardView` for this job's steps; if any are not `DONE`, it self-schedules a 5 s resume timer and pauses. An idle execution stage is a paused workflow, not a busy loop.
- **Pause/resume is a checkpoint, not a restart.** When a pause signal arrives during `executionStep`, the workflow emits `JobPaused`, saves current state, and suspends. Executor workflows detect the job is `PAUSED` and stop polling new steps (they complete any currently-claimed step). A resume signal re-enters `executionStep` from the same checkpoint — no steps are replanned or lost.
- **Workflow step timeouts:** `planStep` 60 s, `detailStep` 60 s, `validationStep` 120 s (it fans out three checker calls), `finaliseStep` 60 s, and `ExecutorWorkflow.executeStep` 120 s. The default 5 s timeout would expire mid-LLM-call (Lesson 4).
- **Bounded retry loop.** A `RETRY` verdict resets the named steps to `OPEN` and returns the job to `EXECUTING`, but only once (`retryCount < 1`); a second `RETRY` accepts the batch and proceeds to finalise, so the pipeline always terminates.
- **The output guardrail can stall, not crash.** If the G1 before-agent-response guardrail refuses the `JobSummary`, `finaliseStep` records the block and ends with the job left `APPROVED`; nothing is finalised and the reason is visible in the UI.
- **Release for liveness:** `StuckStepMonitor` returns a step claimed-but-idle for more than three minutes to `OPEN`, so an executor that fails mid-step does not strand the board. `release` is a no-op unless the step is `CLAIMED`.
- **The step eval signal is downstream and non-blocking.** `StepEvalConsumer` subscribes to `JobEntity` events and records a `StepSignal` after a stage result lands; it never gates the pipeline.
- **Audit review is on the loop.** `recordAuditReview` is accepted only when the job is `COMPLETED` and never changes that status.
- **Idempotency:** deterministic `stepId = jobId + "-p" + index` makes `createStep` idempotent if `detailStep` is retried; `jobId` is the `ProcessOrchestrationWorkflow` id so a redelivered `JobSubmitted` starts the same workflow, not a duplicate.
