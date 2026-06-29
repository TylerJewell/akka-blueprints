# PLAN — task-insight-memory

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Exec[ExecutorAgent]:::agent
  Eval[EvaluatorAgent]:::agent

  WF[MemoryWorkflow]:::wf
  Memory[MemoryEntity]:::ese
  Task[TaskEntity]:::ese
  Queue[TaskQueue]:::ese
  MemView[MemoryView]:::view
  TaskView[TaskView]:::view
  Consumer[TaskConsumer]:::cons
  Sim[TaskSimulator]:::ta
  Drift[DriftWatcher]:::ta
  API[MemoryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit task| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|retrieve insights| MemView
  WF -->|execute task| Exec
  WF -->|evaluate result| Eval
  WF -->|emit events| Task
  WF -->|persist insight| Memory
  Memory -.->|projects| MemView
  Task -.->|projects| TaskView
  API -->|query / SSE| TaskView
  API -->|query| MemView
  Drift -.->|every 5 min| Memory
```

## Interaction sequence — J1 (verified persistence on first attempt)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as MemoryEndpoint
  participant Q as TaskQueue
  participant C as TaskConsumer
  participant W as MemoryWorkflow
  participant MV as MemoryView
  participant EX as ExecutorAgent
  participant EV as EvaluatorAgent
  participant T as TaskEntity
  participant M as MemoryEntity
  participant TV as TaskView

  U->>API: POST /api/tasks {taskType, description}
  API->>Q: append TaskSubmitted
  API-->>U: 202 {taskId}
  Q->>C: TaskSubmitted
  C->>W: start({taskId, taskType, description})
  W->>T: emit TaskCreated (PENDING)

  W->>MV: getInsightsByTaskType(taskType, max=5)
  MV-->>W: [] (first run, no prior insights)
  W->>T: emit TaskExecutionStarted (EXECUTING)
  W->>EX: EXECUTE_TASK(taskType, description, [])
  EX-->>W: TaskResult{answer, confidence=0.82, keyFindings=[...]}
  W->>T: emit TaskResultRecorded

  Note over W: evaluateStep
  W->>EV: EVALUATE_RESULT(TaskResult, acceptanceCriteria)
  EV-->>W: EvalVerdict{VERIFIED, qualityScore=0.88, notes}
  W->>T: emit TaskEvalVerdictRecorded

  Note over W: sanitizeStep (PII scrub, pure function)
  Note over W: persistStep
  W->>M: emit InsightPersisted{taskType, sanitizedText, VERIFIED_EXPERIENCE}
  W->>T: emit TaskVerified (VERIFIED)
  T-->>TV: project
  M-->>MV: project
  TV-->>U: SSE update
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> EXECUTING: TaskExecutionStarted
  EXECUTING --> EVALUATED: TaskResultRecorded + TaskEvalVerdictRecorded
  EVALUATED --> VERIFIED: EvalOutcome = VERIFIED AND confidence >= threshold
  EVALUATED --> FAILED: EvalOutcome = REJECTED OR confidence < threshold
  EXECUTING --> FAILED: defaultStepRecovery failover
  VERIFIED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskCreated : emits
  TaskEntity ||--o{ TaskExecutionStarted : emits
  TaskEntity ||--o{ TaskResultRecorded : emits
  TaskEntity ||--o{ TaskEvalVerdictRecorded : emits
  TaskEntity ||--o{ TaskVerified : emits
  TaskEntity ||--o{ TaskFailed : emits
  TaskView }o--|| TaskEntity : projects
  MemoryEntity ||--o{ InsightPersisted : emits
  MemoryEntity ||--o{ InsightSuperseded : emits
  MemoryEntity ||--o{ CorrectionApplied : emits
  MemoryEntity ||--o{ DemonstrationAdded : emits
  MemoryEntity ||--o{ DriftAssessmentRecorded : emits
  MemoryView }o--|| MemoryEntity : projects
  TaskQueue ||--o{ TaskSubmitted : emits
  TaskConsumer }o--|| TaskQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ExecutorAgent` | `application/ExecutorAgent.java` |
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `MemoryTasks` | `application/MemoryTasks.java` |
| `MemoryWorkflow` | `application/MemoryWorkflow.java` |
| `MemoryEntity` | `application/MemoryEntity.java` (state in `domain/MemoryStore.java`, events in `domain/MemoryEvent.java`) |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/TaskRecord.java`, events in `domain/TaskEvent.java`) |
| `TaskQueue` | `application/TaskQueue.java` |
| `MemoryView` | `application/MemoryView.java` |
| `TaskView` | `application/TaskView.java` |
| `TaskConsumer` | `application/TaskConsumer.java` |
| `TaskSimulator` | `application/TaskSimulator.java` |
| `DriftWatcher` | `application/DriftWatcher.java` |
| `MemoryEndpoint` | `api/MemoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `executeStep` and `evaluateStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — any unrecoverable agent failure ends in `FAILED` rather than a hung workflow.
- **Sanitize step:** `sanitizeStep` is pure-function (no LLM call); it applies regex patterns to each key finding and replaces matches with `[REDACTED]`. The redaction count is included in the `SanitizerApplied` signal for observability.
- **MemoryEntity atomicity:** the memory store is written in a single `InsightPersisted` event per verified result. The workflow never writes partial state to `MemoryEntity`.
- **DriftWatcher idempotency:** the watcher keys its `recordDriftAssessment` call on the 5-minute window timestamp, so a tick that fires twice within the same window produces exactly one event.
- **Insight retrieval ordering:** `getInsightsByTaskType` orders by `persistedAt DESC`; the top 5 most-recent verified insights are passed to `ExecutorAgent`. Superseded insights are filtered out at the view query level.
- **Correction / demonstration writes:** `POST /api/memory/correction` and `POST /api/memory/demonstration` write directly to `MemoryEntity` via `MemoryEndpoint`; they do not go through the workflow. The `InsightProvenance` field distinguishes them from `VERIFIED_EXPERIENCE` insights.
