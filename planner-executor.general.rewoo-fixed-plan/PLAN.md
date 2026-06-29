# PLAN — rewoo-fixed-plan

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
  Worker[WorkerAgent]:::agent
  Solver[SolverAgent]:::agent

  WF[QueryWorkflow]:::wf
  QEntity[QueryEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RequestQueue]:::ese
  View[QueryView]:::view
  Consumer[QueryRequestConsumer]:::cons
  Sim[QuerySimulator]:::ta
  Stuck[StuckQueryMonitor]:::ta
  API[QueryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|WRITE_PLAN| Planner
  WF -->|EXECUTE_STEP| Worker
  WF -->|COMPOSE_ANSWER| Solver
  WF -->|emit events| QEntity
  WF -->|poll| Ctrl
  QEntity -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| QEntity
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as QueryEndpoint
  participant Q as RequestQueue
  participant C as QueryRequestConsumer
  participant W as QueryWorkflow
  participant P as PlannerAgent
  participant Wk as WorkerAgent
  participant S as SolverAgent
  participant E as QueryEntity
  participant CTL as SystemControlEntity
  participant V as QueryView

  U->>API: POST /api/queries {query}
  API->>Q: append QuerySubmitted
  API-->>U: 202 {queryId}
  Q->>C: QuerySubmitted
  C->>W: start({queryId, query})
  W->>E: emit QueryCreated (PLANNING)
  W->>P: WRITE_PLAN(query)
  P-->>W: QueryPlan (3-5 steps with #En placeholders)
  W->>E: emit QueryPlanned, status EXECUTING
  loop for each PlanStep in order
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>W: resolve #En references in step.inputExpression
    W->>W: ToolCallGuardrail.vet(step, resolvedInput)
    W->>Wk: EXECUTE_STEP(step, resolvedInput)
    Wk-->>W: ToolResult
    W->>W: SecretScrubber.scrub(content)
    W->>E: emit StepCompleted (StepRecord)
  end
  W->>S: COMPOSE_ANSWER(filledPlan)
  S-->>W: SolvedAnswer
  W->>E: emit QuerySolved
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: QueryPlanned
  EXECUTING --> EXECUTING: StepStarted / StepCompleted
  EXECUTING --> COMPLETED: QuerySolved
  EXECUTING --> FAILED: QueryFailed / StepBlocked
  EXECUTING --> HALTED: QueryHaltedOperator
  EXECUTING --> STUCK: QueryFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QueryPlanned : emits
  QueryEntity ||--o{ StepStarted : emits
  QueryEntity ||--o{ StepBlocked : emits
  QueryEntity ||--o{ StepCompleted : emits
  QueryEntity ||--o{ QuerySolved : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryEntity ||--o{ QueryHaltedOperator : emits
  QueryEntity ||--o{ QueryFailedTimeout : emits
  QueryView }o--|| QueryEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RequestQueue ||--o{ QuerySubmitted : emits
  QueryRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `WorkerAgent` | `application/WorkerAgent.java` |
| `SolverAgent` | `application/SolverAgent.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `QueryView` | `application/QueryView.java` |
| `QueryRequestConsumer` | `application/QueryRequestConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `StuckQueryMonitor` | `application/StuckQueryMonitor.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `WorkerTasks` | `application/WorkerTasks.java` |
| `SolverTasks` | `application/SolverTasks.java` |
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `resolveStep` 15 s, `guardrailStep` 10 s, `executeStep` 90 s (covers any tool simulation plus agent round trip), `solveStep` 60 s. Default recovery: `maxRetries(2).failoverTo(QueryWorkflow::error)`.
- **Fixed-plan discipline:** the `QueryPlan` is immutable once emitted by `QueryPlanned`. The Worker reads steps in index order; it never rewrites the plan. Variable resolution is purely substitutional — it cannot add, remove, or reorder steps.
- **Guardrail is blocking:** a `StepBlocked` event transitions the query directly to `FAILED`. There is no replan branch in this pattern; plan revision would defeat the purpose of reasoning-before-observation.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously. An operator halt arriving mid-`executeStep` lets the tool call finish; the loop exits at the next `checkHaltStep`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure. The same raw tool output always yields the same scrubbed string, making `StepCompleted` events deterministic and replayable.
- **Stuck detection:** `StuckQueryMonitor` ticks every 30 s; queries in `EXECUTING` for more than 5 minutes receive a `QueryFailedTimeout` command.
- **Idempotency:** `QueryEndpoint.submit` deduplicates `POST /api/queries` on `(query, requestedBy)` over a 10 s window.
