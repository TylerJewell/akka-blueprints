# PLAN — custom-orchestration-agent

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
  Disp[ToolDispatcherAgent]:::agent

  WF[TaskWorkflow]:::wf
  Task[TaskEntity]:::ese
  Registry[StrategyRegistry]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RequestQueue]:::ese
  View[TaskView]:::view
  Consumer[TaskRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stuck[StuckTaskMonitor]:::ta
  API[TaskEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|activate strategy| Registry
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|load active strategy| Registry
  WF -->|INITIALIZE_CONTEXT / ROUTE / CONCLUDE| Orch
  WF -->|DISPATCH_TOOL| Disp
  WF -->|emit events| Task
  WF -->|poll halt flag| Ctrl
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
  participant SR as StrategyRegistry
  participant O as OrchestratorAgent
  participant D as ToolDispatcherAgent
  participant E as TaskEntity
  participant CTL as SystemControlEntity
  participant V as TaskView

  U->>API: POST /api/tasks {prompt}
  API->>Q: append TaskSubmitted
  API-->>U: 202 {taskId}
  Q->>C: TaskSubmitted
  C->>W: start({taskId, prompt})
  W->>E: emit TaskCreated (INITIALIZING)
  W->>SR: getActive
  SR-->>W: OrchestrationStrategy "default"
  W->>E: emit TaskInitialized (strategy loaded)
  W->>O: INITIALIZE_CONTEXT(goal, strategy)
  O-->>W: ExecutionContext
  W->>E: emit ContextInitialized (EXECUTING)
  loop until Conclude | Abort | Halt | Budget
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>O: ROUTE(executionContext)
    O-->>W: RoutingDecision(CallTool)
    W->>W: ToolCallGuardrail.vet(decision, strategy)
    W->>D: DISPATCH_TOOL(toolCall)
    D-->>W: ToolResult
    W->>W: SecretScrubber.scrub(content)
    W->>E: emit ToolRecorded (TraceEntry)
  end
  W->>O: CONCLUDE(executionContext)
  O-->>W: TaskAnswer
  W->>E: emit TaskCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> INITIALIZING
  INITIALIZING --> EXECUTING: ContextInitialized
  EXECUTING --> EXECUTING: ToolRecorded / ToolBlocked / ContextRevisited
  EXECUTING --> COMPLETED: TaskCompleted
  EXECUTING --> FAILED: TaskFailed
  EXECUTING --> HALTED: TaskHaltedOperator
  EXECUTING --> STUCK: TaskFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskCreated : emits
  TaskEntity ||--o{ TaskInitialized : emits
  TaskEntity ||--o{ ContextInitialized : emits
  TaskEntity ||--o{ ToolDispatched : emits
  TaskEntity ||--o{ ToolBlocked : emits
  TaskEntity ||--o{ ToolRecorded : emits
  TaskEntity ||--o{ ContextRevisited : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskEntity ||--o{ TaskFailed : emits
  TaskEntity ||--o{ TaskHaltedOperator : emits
  TaskEntity ||--o{ TaskFailedTimeout : emits
  TaskView }o--|| TaskEntity : projects
  StrategyRegistry ||--o{ StrategyRegistered : emits
  StrategyRegistry ||--o{ StrategyActivated : emits
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RequestQueue ||--o{ TaskSubmitted : emits
  TaskRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OrchestratorAgent` | `application/OrchestratorAgent.java` |
| `ToolDispatcherAgent` | `application/ToolDispatcherAgent.java` |
| `TaskWorkflow` | `application/TaskWorkflow.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `StrategyRegistry` | `application/StrategyRegistry.java` |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `TaskView` | `application/TaskView.java` |
| `TaskRequestConsumer` | `application/TaskRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StuckTaskMonitor` | `application/StuckTaskMonitor.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `OrchestratorTasks` | `application/OrchestratorTasks.java` |
| `DispatcherTasks` | `application/DispatcherTasks.java` |
| `TaskEndpoint` | `api/TaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `initContextStep` 45 s, `routeStep` 45 s, `dispatchStep` 90 s, `concludeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(TaskWorkflow::error)`.
- **Iteration budget:** `strategy.maxRouteIterations` caps the number of `CallTool` ticks; exceeding it transitions to `failStep` with reason `"iteration budget exhausted"`.
- **Revisit budget:** `strategy.maxRevisits` caps the number of `Revisit` decisions; exceeding it forces the next route tick to skip `Revisit` options.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight tool call finish; the loop exits at the next `checkHaltStep`.
- **Strategy isolation:** a `TaskWorkflow` reads the strategy once in `loadStrategyStep` and holds it in workflow state. Swapping the active strategy mid-task does not affect in-flight workflows.
- **Idempotency:** `TaskEndpoint.submit` deduplicates `(prompt, requestedBy)` over a 10 s window.
- **Stuck detection:** `StuckTaskMonitor` ticks every 30 s; tasks `EXECUTING` for > 5 minutes are marked `STUCK`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; the same input always yields the same scrubbed output, keeping `TraceEntry` events deterministic and replayable.
