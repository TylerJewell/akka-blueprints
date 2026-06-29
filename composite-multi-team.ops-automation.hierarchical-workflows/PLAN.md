# PLAN — hierarchical-workflow-automation

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

  Orch[Orchestrator]:::agent
  DLead[DiscoveryLead]:::agent
  Scan[SystemScanner]:::agent
  ELead[ExecutionLead]:::agent
  Exec[TaskExecutor]:::agent
  Val[Validator]:::agent

  WInst[WorkflowInstance]:::wf
  ExecWF[ExecutorWorkflow]:::wf

  WEntity[WorkflowEntity]:::ese
  TEntity[TaskEntity]:::ese
  RQueue[RequestQueue]:::ese
  WBoard[WorkflowBoardView]:::view
  TBoard[TaskBoardView]:::view
  ReqC[RequestConsumer]:::cons
  AuditC[AuditEvalConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stuck[StuckTaskMonitor]:::ta
  API[WorkflowEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit request| RQueue
  Sim -.->|every 60s| RQueue
  RQueue -.->|subscribes| ReqC
  ReqC -->|create| WEntity
  ReqC -->|start workflow| WInst

  WInst -->|PLAN / REPORT| Orch
  WInst -->|PLAN_DISCOVERY / SYNTHESIZE| DLead
  WInst -->|SCAN x N| Scan
  WInst -->|PLAN_EXECUTION| ELead
  WInst -->|VALIDATE x axes| Val
  WInst -->|create one per task| TEntity
  WInst -->|record stages| WEntity
  WInst -->|poll tasks| TBoard

  ExecWF -->|poll board| TBoard
  ExecWF -->|atomic claim| TEntity
  ExecWF -->|EXECUTE_TASK| Exec

  Scan -.->|SystemTools write| WEntity
  Exec -.->|SystemTools write| TEntity
  Val -.->|SystemTools write| WEntity

  WEntity -.->|projects| WBoard
  TEntity -.->|projects| TBoard
  WEntity -.->|stage events| AuditC
  AuditC -->|record eval| WEntity
  Stuck -.->|every 30s| TEntity

  API -->|query / SSE| WBoard
  API -->|query / SSE| TBoard
  API -->|compliance review| WEntity
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions, scheduled ticks, and guarded tool writes. `SystemScanner`, `TaskExecutor`, and `Validator` are each one agent class run as several instances — scanner instances per target system, `executor-1`/`executor-2`, and one validator per axis (`correctness`, `safety`, `compliance`). The top-level `WorkflowInstance` is the orchestrator; the three teams each run a different internal coordination capability: discovery delegates and fans in, execution is a team over the shared `TaskBoardView`, validation is a moderated panel feeding `AggregationRule`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as WorkflowEndpoint
  participant Q as RequestQueue
  participant C as RequestConsumer
  participant WI as WorkflowInstance
  participant OR as Orchestrator
  participant DL as DiscoveryLead
  participant SC as SystemScanner
  participant T as TaskEntity
  participant EW as ExecutorWorkflow
  participant TE as TaskExecutor
  participant VL as Validator
  participant WE as WorkflowEntity

  U->>API: POST /api/requests {description}
  API->>Q: submitRequest(opsRequest)
  API-->>U: 202 {requestId}
  Q->>C: RequestSubmitted
  C->>WE: createWorkflow
  C->>WI: start({requestId})
  WI->>OR: PLAN(description)
  OR-->>WI: WorkflowPlan
  WI->>WE: recordPlan
  WI->>DL: PLAN_DISCOVERY
  DL-->>WI: DiscoveryPlan{scanTargets}
  WI->>SC: SCAN(target) x N
  SC-->>WE: appendScanResult (guarded)
  WI->>DL: SYNTHESIZE(results)
  DL-->>WI: DiscoverySummary
  WI->>WE: recordDiscovery
  WI->>WE: stage event -> AuditEvalConsumer records eval
  WI->>ELead: PLAN_EXECUTION
  Note over WI: writes one TaskEntity per task (OPEN)
  WI->>T: createTask x M
  Note over EW: executor loops already polling the board
  EW->>T: claim(executor-1) (atomic, single winner)
  EW->>TE: EXECUTE_TASK(task)
  TE-->>T: executeTask result (guarded)
  EW->>T: recordDone
  Note over WI: poll until all tasks DONE
  WI->>WE: recordExecution
  WI->>VL: VALIDATE(axis, summary) x 3
  VL-->>WE: appendValidationNote (guarded)
  WI->>WI: AggregationRule(notes) -> PASS
  WI->>WE: recordValidation(PASS)
  WI->>OR: REPORT(summary)
  Note over OR: before-agent-response guardrail vets OpsReport
  OR-->>WI: OpsReport
  WI->>WE: complete(report, url)
  WE-->>API: SSE status=COMPLETED
```

## State machine — `WorkflowEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED: createWorkflow
  SUBMITTED --> PLANNED: recordPlan
  PLANNED --> DISCOVERING: discovery begins
  DISCOVERING --> DISCOVERED: recordDiscovery
  DISCOVERED --> EXECUTING: recordTasks
  EXECUTING --> EXECUTED: recordExecution (all tasks done)
  EXECUTED --> VALIDATING: panel runs
  VALIDATING --> VALIDATED: recordValidation PASS
  VALIDATING --> EXECUTING: requestRetry (one bounded round)
  VALIDATED --> COMPLETED: complete (output guardrail passes)
  VALIDATED --> VALIDATED: recordReportBlock (guardrail refusal)
  COMPLETED --> COMPLETED: recordComplianceReview (post-execution, non-blocking)
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  WorkflowEntity ||--o{ WorkflowCreated : emits
  WorkflowEntity ||--o{ WorkflowPlanned : emits
  WorkflowEntity ||--o{ DiscoverySynthesized : emits
  WorkflowEntity ||--o{ TasksPlanned : emits
  WorkflowEntity ||--o{ ExecutionSummarized : emits
  WorkflowEntity ||--o{ ValidationCompleted : emits
  WorkflowEntity ||--o{ RetryRequested : emits
  WorkflowEntity ||--o{ ReportCompleted : emits
  WorkflowEntity ||--o{ AuditEvaluated : emits
  WorkflowEntity ||--o{ ComplianceReviewRecorded : emits
  WorkflowBoardView }o--|| WorkflowEntity : projects
  AuditEvalConsumer }o--|| WorkflowEntity : subscribes
  WorkflowEntity ||--o{ TaskEntity : "owns M tasks"
  TaskEntity ||--o{ TaskCreated : emits
  TaskEntity ||--o{ TaskClaimed : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskEntity ||--o{ TaskReleased : emits
  TaskBoardView }o--|| TaskEntity : projects
  RequestQueue ||--o{ RequestSubmitted : emits
  RequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `Orchestrator` | `application/Orchestrator.java` |
| `DiscoveryLead` | `application/DiscoveryLead.java` |
| `SystemScanner` | `application/SystemScanner.java` |
| `ExecutionLead` | `application/ExecutionLead.java` |
| `TaskExecutor` | `application/TaskExecutor.java` |
| `Validator` | `application/Validator.java` |
| `WorkflowTasks` | `application/WorkflowTasks.java` |
| `SystemTools` | `application/SystemTools.java` |
| `AggregationRule` | `application/AggregationRule.java` |
| `ResultEvaluator` | `application/ResultEvaluator.java` |
| `WorkflowInstance` | `application/WorkflowInstance.java` |
| `ExecutorWorkflow` | `application/ExecutorWorkflow.java` |
| `WorkflowEntity` | `application/WorkflowEntity.java` (state in `domain/WorkflowState.java`, events in `domain/WorkflowEvent.java`) |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `WorkflowBoardView` | `application/WorkflowBoardView.java` |
| `TaskBoardView` | `application/TaskBoardView.java` |
| `RequestConsumer` | `application/RequestConsumer.java` |
| `AuditEvalConsumer` | `application/AuditEvalConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StuckTaskMonitor` | `application/StuckTaskMonitor.java` |
| `WorkflowEndpoint` | `api/WorkflowEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **6 autonomous-agent · 2 workflow · 3 event-sourced-entity · 2 view · 2 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Two coordination primitives sit under one pipeline.** The top-level `WorkflowInstance` is a sequential delegation: each stage runs and writes its result onto the shared `WorkflowEntity` before the next begins. The execution stage hands off to an independent team: the workflow seeds the board and then waits, while the per-executor `ExecutorWorkflow` loops claim and carry out tasks on their own clock.
- **Atomic claim is the execution-team primitive.** `TaskEntity` is a single-writer; `claim(executorId)` emits `TaskClaimed` only when the current status is `OPEN`. Two executor workflows that read the same `OPEN` task from the board and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `Task` and returns to polling. No lock, no external queue.
- **The execution wait is a poll, not a block.** `WorkflowInstance.executionStep` queries `TaskBoardView` for this request's tasks; if any are not `DONE`, it self-schedules a 5 s resume timer and pauses. An idle execution stage is a paused workflow, not a busy loop.
- **Workflow step timeouts:** every step that calls an agent sets an explicit `stepTimeout` (Lesson 4) — `planStep` 60 s, `discoveryStep` 120 s (it fans out several scanner calls), `validationStep` 90 s, `reportStep` 60 s, and `ExecutorWorkflow.executeStep` 90 s.
- **Bounded retry loop.** A `RETRY` verdict resets the named tasks to `OPEN` and returns the workflow to `EXECUTING`, but only once (`retryCount < 1`); a second `RETRY` accepts the summary and proceeds to reporting, so the pipeline always terminates.
- **The output guardrail can stall, not crash.** If the G1 before-agent-response guardrail refuses the final report, `reportStep` records the block and ends with the workflow left `VALIDATED`; nothing is completed and the reason is visible in the UI.
- **Release for liveness:** `StuckTaskMonitor` returns a task claimed-but-idle for more than two minutes to `OPEN`, so an executor that fails mid-task does not strand the board. `release` is a no-op unless the task is `CLAIMED`.
- **The audit eval is downstream and non-blocking.** `AuditEvalConsumer` subscribes to `WorkflowEntity` events and records an `AuditEval` after a stage result lands; it never gates the pipeline (control E1).
- **Compliance review is on the loop.** `recordComplianceReview` is accepted only when the workflow is `COMPLETED` and never changes that status (control HO1).
- **Idempotency:** deterministic `taskId = requestId + "-t" + index` makes `createTask` idempotent if `executionStep` is retried; `requestId` is the `WorkflowInstance` id so a redelivered `RequestSubmitted` starts the same workflow, not a duplicate.
