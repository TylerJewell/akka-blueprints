# PLAN — llm-compiler-dag

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

  Planner[PlannerAgent]:::agent
  Joiner[JoinerAgent]:::agent

  TFU[TaskFetchingUnit]:::wf
  Job[JobEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RequestQueue]:::ese
  View[JobView]:::view
  Consumer[JobRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stale[StaleJobMonitor]:::ta
  API[JobEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| TFU
  TFU -->|COMPILE| Planner
  TFU -->|SEARCH parallel| TFU
  TFU -->|CALCULATOR parallel| TFU
  TFU -->|LOOKUP parallel| TFU
  TFU -->|CODE_EVAL parallel| TFU
  TFU -->|JOIN| Joiner
  TFU -->|emit events| Job
  TFU -->|poll| Ctrl
  Job -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Job
```

## Interaction sequence — J1 (happy path with parallel dispatch)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as JobEndpoint
  participant Q as RequestQueue
  participant C as JobRequestConsumer
  participant W as TaskFetchingUnit
  participant P as PlannerAgent
  participant T1 as Tool Branch A
  participant T2 as Tool Branch B
  participant E as JobEntity
  participant CTL as SystemControlEntity
  participant J as JoinerAgent
  participant V as JobView

  U->>API: POST /api/jobs {query}
  API->>Q: append QuerySubmitted
  API-->>U: 202 {jobId}
  Q->>C: QuerySubmitted
  C->>W: start({jobId, query})
  W->>E: emit JobCreated (COMPILING)
  W->>P: COMPILE(query)
  P-->>W: CompilationPlan (DAG of ToolCall nodes)
  W->>E: emit JobCompiled, status RUNNING
  W->>CTL: get halt flag
  CTL-->>W: halted=false
  W->>W: guardStep.vet(frontierNodes)
  par parallel dispatch
    W->>T1: SEARCH("akka 3.6.0 release")
    W->>T2: CALCULATOR("42 * 17")
  end
  T1-->>W: ToolResult(callId=c1, ok=true)
  T2-->>W: ToolResult(callId=c2, ok=true)
  W->>W: SecretScrubber.scrub(each output)
  W->>E: emit ToolCallResolved x2
  Note over W: frontier re-evaluated; c3 depends on c1 ✓
  W->>W: guardStep.vet([c3])
  W->>T1: LOOKUP("akka-version-minor")
  T1-->>W: ToolResult(callId=c3, ok=true)
  W->>W: SecretScrubber.scrub(output)
  W->>E: emit ToolCallResolved
  Note over W: frontier empty → join
  W->>J: JOIN(ResultSet)
  J-->>W: QueryAnswer
  W->>E: emit JobCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `JobEntity`

```mermaid
stateDiagram-v2
  [*] --> COMPILING
  COMPILING --> RUNNING: JobCompiled
  RUNNING --> RUNNING: ToolCallResolved / ToolCallBlocked / ToolCallFailed
  RUNNING --> COMPLETED: JobCompleted
  RUNNING --> FAILED: JobFailed
  RUNNING --> HALTED: JobHaltedOperator
  RUNNING --> STALE: JobFailedTimeout
  STALE --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  JobEntity ||--o{ JobCompiled : emits
  JobEntity ||--o{ ToolCallDispatched : emits
  JobEntity ||--o{ ToolCallBlocked : emits
  JobEntity ||--o{ ToolCallResolved : emits
  JobEntity ||--o{ ToolCallFailed : emits
  JobEntity ||--o{ JobCompleted : emits
  JobEntity ||--o{ JobFailed : emits
  JobEntity ||--o{ JobHaltedOperator : emits
  JobEntity ||--o{ JobFailedTimeout : emits
  JobView }o--|| JobEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RequestQueue ||--o{ QuerySubmitted : emits
  JobRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `JoinerAgent` | `application/JoinerAgent.java` |
| `TaskFetchingUnit` | `application/TaskFetchingUnit.java` |
| `JobEntity` | `application/JobEntity.java` (state in `domain/Job.java`, events in `domain/JobEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `JobView` | `application/JobView.java` |
| `JobRequestConsumer` | `application/JobRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StaleJobMonitor` | `application/StaleJobMonitor.java` |
| `DispatchGuardrail` | `application/DispatchGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `JoinerTasks` | `application/JoinerTasks.java` |
| `JobEndpoint` | `api/JobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Parallel dispatch:** `parallelDispatchStep` fans out one Akka workflow branch per approved frontier node. All branches run concurrently; the step waits for the last to complete before sanitize.
- **DAG frontier rule:** a node is eligible when every id in its `dependsOn` list is present in `resolvedIds`. The frontier set is recomputed after every batch of `ToolCallResolved` events.
- **Guardrail and skipped nodes:** a `SKIPPED` node's `callId` is added to `resolvedIds` immediately so dependent nodes that do not strictly need the skipped output can still be dispatched. Dependents that required the skipped output are treated as `FAILED`.
- **Step timeouts:** `compileStep` 60 s, `guardStep` 30 s, `parallelDispatchStep` 120 s, `joinStep` 60 s. Default recovery: `maxRetries(2).failoverTo(TaskFetchingUnit::error)`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `parallelDispatchStep` lets the entire current batch finish; the loop exits at the next `checkHaltStep`.
- **Failure budget:** if the error count for a single job exceeds 3 `ToolCallFailed` events, the workflow transitions to `failStep`.
- **Stale detection:** `StaleJobMonitor` ticks every 30 s; `JobFailedTimeout` is non-fatal to other jobs. The workflow's `frontierStep` checks the entity's status and exits when `status == STALE`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; the same input always yields the same scrubbed output, keeping events deterministic and replayable.
