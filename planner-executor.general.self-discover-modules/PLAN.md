# PLAN — self-discover-modules

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

  Struct[StructurerAgent]:::agent
  Solver[SolverAgent]:::agent

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
  WF -->|SELECT_MODULES / COMPOSE_STRUCTURE / REVISE_STRUCTURE| Struct
  WF -->|EXECUTE_STEP / COMPOSE_ANSWER| Solver
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
  participant ST as StructurerAgent
  participant EV as StructureEvaluator
  participant SV as SolverAgent
  participant E as TaskEntity
  participant CTL as SystemControlEntity
  participant V as TaskView

  U->>API: POST /api/tasks {prompt}
  API->>Q: append TaskSubmitted
  API-->>U: 202 {taskId}
  Q->>C: TaskSubmitted
  C->>W: start({taskId, prompt})
  W->>E: emit TaskCreated (STRUCTURING)
  W->>ST: SELECT_MODULES(prompt)
  ST-->>W: List<String> selectedModules
  W->>ST: COMPOSE_STRUCTURE(prompt, selectedModules)
  ST-->>W: ReasoningStructure
  W->>E: emit StructureComposed
  W->>EV: evaluate(structure)
  EV-->>W: EvalResult{passed=true, score=0.85}
  W->>E: emit StructureApproved, status SOLVING
  loop for each ModuleStep in structure
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>SV: EXECUTE_STEP(moduleName, adaptedDescription, accumulated)
    SV-->>W: StepExecution
    W->>E: emit StepExecuted
  end
  W->>SV: COMPOSE_ANSWER(stepExecutions)
  SV-->>W: TaskAnswer
  W->>E: emit TaskCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> STRUCTURING
  STRUCTURING --> STRUCTURING: StructureComposed / StructureRejected / StructureRevised
  STRUCTURING --> SOLVING: StructureApproved
  SOLVING --> SOLVING: StepExecuted
  SOLVING --> COMPLETED: TaskCompleted
  SOLVING --> FAILED: TaskFailed
  SOLVING --> HALTED: TaskHaltedOperator
  STRUCTURING --> FAILED: TaskFailed
  STRUCTURING --> STUCK: TaskFailedTimeout
  SOLVING --> STUCK: TaskFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ StructureComposed : emits
  TaskEntity ||--o{ StructureRejected : emits
  TaskEntity ||--o{ StructureApproved : emits
  TaskEntity ||--o{ StructureRevised : emits
  TaskEntity ||--o{ StepExecuted : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskEntity ||--o{ TaskFailed : emits
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
| `StructurerAgent` | `application/StructurerAgent.java` |
| `SolverAgent` | `application/SolverAgent.java` |
| `TaskWorkflow` | `application/TaskWorkflow.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `TaskView` | `application/TaskView.java` |
| `TaskRequestConsumer` | `application/TaskRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StuckTaskMonitor` | `application/StuckTaskMonitor.java` |
| `StructureEvaluator` | `application/StructureEvaluator.java` |
| `StructurerTasks` | `application/StructurerTasks.java` |
| `SolverTasks` | `application/SolverTasks.java` |
| `TaskEndpoint` | `api/TaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `selectStep` 45 s, `composeStep` 60 s, `reviseStep` 60 s, `executeStepN` 90 s (covers a multi-sentence observation from the Solver), `composeAnswerStep` 60 s. Default recovery: `maxRetries(2).failoverTo(TaskWorkflow::error)`.
- **Revision budget:** the Structurer may revise the structure at most twice before the workflow transitions to `failStep` on a third rejection.
- **Step iteration:** the workflow iterates through `structure.adaptedSteps` in index order. Each iteration is a separate workflow step so that a halt check occurs between steps.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during an `executeStepN` lets the in-flight step finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `TaskEndpoint.submit` uses `(prompt, requestedBy)` over a 10 s window to dedupe `POST /api/tasks`.
- **Stuck detection:** `StuckTaskMonitor` ticks every 30 s; tasks in `STRUCTURING` or `SOLVING` for > 5 minutes are marked `STUCK`.
- **Evaluator determinism:** `StructureEvaluator.evaluate` is pure — same structure always yields the same score and rejection reason, keeping events deterministic and replayable.
