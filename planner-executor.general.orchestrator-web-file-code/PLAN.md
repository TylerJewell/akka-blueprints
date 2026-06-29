# PLAN — orchestrator-web-file-code

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  Orch[OrchestratorAgent]:::agent
  Web[WebSurferAgent]:::agent
  File[FileSurferAgent]:::agent
  Coder[CoderAgent]:::agent
  Term[TerminalAgent]:::agent

  WF[TaskWorkflow]:::wf
  Task[TaskEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RequestQueue]:::ese
  View[TaskView]:::view
  Consumer[TaskRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stuck[StuckTaskMonitor]:::ta
  API[TaskEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE| Orch
  WF -->|WEB_SEARCH| Web
  WF -->|FILE_READ| File
  WF -->|WRITE_CODE| Coder
  WF -->|RUN_COMMAND| Term
  WF -->|emit events| Task
  WF -->|poll| Ctrl
  Task -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Task
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as TaskEndpoint
  participant Q as RequestQueue
  participant C as TaskRequestConsumer
  participant W as TaskWorkflow
  participant O as OrchestratorAgent
  participant S as Specialist (Web/File/Coder/Terminal)
  participant E as TaskEntity
  participant CTL as SystemControlEntity
  participant V as TaskView

  U->>API: POST /api/tasks {prompt}
  API->>Q: append TaskSubmitted
  API-->>U: 202 {taskId}
  Q->>C: TaskSubmitted
  C->>W: start({taskId, prompt})
  W->>E: emit TaskCreated (PLANNING)
  W->>O: PLAN(prompt)
  O-->>W: TaskLedger
  W->>E: emit TaskPlanned, status EXECUTING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>O: DECIDE(ledgers)
    O-->>W: Continue(DispatchDecision)
    W->>W: guardrail.vet(decision)
    W->>S: runSingleTask(subtask)
    S-->>W: SubtaskResult
    W->>W: SecretScrubber.scrub(content)
    W->>E: emit SubtaskRecorded (ProgressEntry)
    W->>W: SafetyEvaluator.evaluate(entry)
  end
  W->>O: COMPOSE_ANSWER
  O-->>W: TaskAnswer
  W->>E: emit TaskCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: TaskPlanned
  EXECUTING --> EXECUTING: SubtaskRecorded / SubtaskBlocked / LedgerRevised
  EXECUTING --> COMPLETED: TaskCompleted
  EXECUTING --> FAILED: TaskFailed
  EXECUTING --> HALTED: TaskHaltedOperator / TaskHaltedAutomatic
  EXECUTING --> STUCK: TaskFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskPlanned : emits
  TaskEntity ||--o{ SubtaskDispatched : emits
  TaskEntity ||--o{ SubtaskBlocked : emits
  TaskEntity ||--o{ SubtaskRecorded : emits
  TaskEntity ||--o{ LedgerRevised : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskEntity ||--o{ TaskFailed : emits
  TaskEntity ||--o{ TaskHaltedAutomatic : emits
  TaskEntity ||--o{ TaskHaltedOperator : emits
  TaskEntity ||--o{ TaskFailedTimeout : emits
  TaskView }o--|| TaskEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RequestQueue ||--o{ TaskSubmitted : emits
  TaskRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OrchestratorAgent` | `application/OrchestratorAgent.java` |
| `WebSurferAgent` | `application/WebSurferAgent.java` |
| `FileSurferAgent` | `application/FileSurferAgent.java` |
| `CoderAgent` | `application/CoderAgent.java` |
| `TerminalAgent` | `application/TerminalAgent.java` |
| `TaskWorkflow` | `application/TaskWorkflow.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `TaskView` | `application/TaskView.java` |
| `TaskRequestConsumer` | `application/TaskRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StuckTaskMonitor` | `application/StuckTaskMonitor.java` |
| `DispatchGuardrail` | `application/DispatchGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `SafetyEvaluator` | `application/SafetyEvaluator.java` |
| `OrchestratorTasks` | `application/OrchestratorTasks.java` |
| `SpecialistTasks` | `application/SpecialistTasks.java` |
| `TaskEndpoint` | `api/TaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any specialist call, including a generous slack for a slow LLM), `decideStep` 45 s, `completeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(TaskWorkflow::error)`.
- **Replan budget:** the orchestrator may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the orchestrator may emit `Continue` on the same `(specialist, subtask)` at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight subtask finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `TaskEndpoint.submit` uses `(prompt, requestedBy)` over a 10 s window to dedupe `POST /api/tasks`.
- **Stuck detection:** `StuckTaskMonitor` ticks every 30 s; `TaskFailedTimeout` is non-fatal to other tasks. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, which keeps `ProgressEntry` events deterministic and replayable.
