# PLAN — plumber-data-engineering-assistant

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

  Planner[PipelinePlannerAgent]:::agent
  Spark[SparkAgent]:::agent
  Beam[BeamAgent]:::agent
  Dbt[DbtAgent]:::agent
  Schema[SchemaAgent]:::agent

  WF[PipelineWorkflow]:::wf
  Pipe[PipelineEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RequestQueue]:::ese
  View[PipelineView]:::view
  Consumer[PipelineRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stuck[StuckPipelineMonitor]:::ta
  API[PipelineEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE_DEFINITION| Planner
  WF -->|GENERATE_SPARK| Spark
  WF -->|GENERATE_BEAM| Beam
  WF -->|GENERATE_DBT| Dbt
  WF -->|VALIDATE_SCHEMA| Schema
  WF -->|emit events| Pipe
  WF -->|poll| Ctrl
  Pipe -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Pipe
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PipelineEndpoint
  participant Q as RequestQueue
  participant C as PipelineRequestConsumer
  participant W as PipelineWorkflow
  participant P as PipelinePlannerAgent
  participant S as Specialist (Spark/Beam/dBT/Schema)
  participant E as PipelineEntity
  participant CTL as SystemControlEntity
  participant V as PipelineView

  U->>API: POST /api/pipelines {prompt}
  API->>Q: append PipelineSubmitted
  API-->>U: 202 {pipelineId}
  Q->>C: PipelineSubmitted
  C->>W: start({pipelineId, prompt})
  W->>E: emit PipelineCreated (PLANNING)
  W->>P: PLAN(prompt)
  P-->>W: PipelineLedger
  W->>E: emit PipelinePlanned, status EXECUTING
  loop until Finalise | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE(ledgers)
    P-->>W: Continue(StageDecision)
    W->>S: runSingleTask(spec)
    S-->>W: StageResult
    W->>W: PipelineTestGate.validate(stageResult)
    W->>E: emit StageRecorded (ProgressEntry)
  end
  W->>P: COMPOSE_DEFINITION
  P-->>W: PipelineDefinition
  W->>E: emit PipelineFinalised
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `PipelineEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: PipelinePlanned
  EXECUTING --> EXECUTING: StageRecorded / StageBlocked / LedgerRevised
  EXECUTING --> FINALISED: PipelineFinalised
  EXECUTING --> FAILED: PipelineFailed
  EXECUTING --> HALTED: PipelineHaltedOperator
  EXECUTING --> STUCK: PipelineFailedTimeout
  STUCK --> [*]
  FINALISED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  PipelineEntity ||--o{ PipelinePlanned : emits
  PipelineEntity ||--o{ StageDispatched : emits
  PipelineEntity ||--o{ StageBlocked : emits
  PipelineEntity ||--o{ StageRecorded : emits
  PipelineEntity ||--o{ LedgerRevised : emits
  PipelineEntity ||--o{ PipelineFinalised : emits
  PipelineEntity ||--o{ PipelineFailed : emits
  PipelineEntity ||--o{ PipelineHaltedOperator : emits
  PipelineEntity ||--o{ PipelineFailedTimeout : emits
  PipelineView }o--|| PipelineEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RequestQueue ||--o{ PipelineSubmitted : emits
  PipelineRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PipelinePlannerAgent` | `application/PipelinePlannerAgent.java` |
| `SparkAgent` | `application/SparkAgent.java` |
| `BeamAgent` | `application/BeamAgent.java` |
| `DbtAgent` | `application/DbtAgent.java` |
| `SchemaAgent` | `application/SchemaAgent.java` |
| `PipelineWorkflow` | `application/PipelineWorkflow.java` |
| `PipelineEntity` | `application/PipelineEntity.java` (state in `domain/Pipeline.java`, events in `domain/PipelineEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `PipelineView` | `application/PipelineView.java` |
| `PipelineRequestConsumer` | `application/PipelineRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StuckPipelineMonitor` | `application/StuckPipelineMonitor.java` |
| `PipelineTestGate` | `application/PipelineTestGate.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `EngineTasks` | `application/EngineTasks.java` |
| `PipelineEndpoint` | `api/PipelineEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (generous slack for a slow LLM), `testGateStep` 30 s, `decideStep` 45 s, `composeDefinitionStep` 60 s. Default recovery: `maxRetries(2).failoverTo(PipelineWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same `(engine, stageKind)` at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight stage finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `PipelineEndpoint.submit` uses `(prompt, requestedBy)` over a 10 s window to dedupe `POST /api/pipelines`.
- **Stuck detection:** `StuckPipelineMonitor` ticks every 30 s; `PipelineFailedTimeout` is non-fatal to other pipelines. The workflow's `decideStep` checks the entity's status and exits when it reads `STUCK`.
- **Secret scrubber determinism:** `SecretScrubber.scrub` is pure; the same input always yields the same output, keeping `ProgressEntry` events deterministic and replayable.
