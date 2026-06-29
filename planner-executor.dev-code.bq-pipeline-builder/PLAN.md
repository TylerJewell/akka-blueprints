# PLAN — bq-pipeline-builder

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
  Schema[SchemaAnalystAgent]:::agent
  Sql[SqlComposerAgent]:::agent
  Dataform[DataformModelerAgent]:::agent
  Validator[ValidatorAgent]:::agent

  WF[PipelineWorkflow]:::wf
  Pipeline[PipelineEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[PipelineQueue]:::ese
  View[PipelineView]:::view
  Consumer[PipelineRequestConsumer]:::cons
  Sim[PipelineSimulator]:::ta
  Stale[StalePipelineMonitor]:::ta
  API[PipelineEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_PIPELINE / DECIDE_STEP / COMPOSE_MANIFEST| Planner
  WF -->|ANALYZE_SCHEMA| Schema
  WF -->|COMPOSE_SQL| Sql
  WF -->|MODEL_DATAFORM| Dataform
  WF -->|VALIDATE_OUTPUT| Validator
  WF -->|emit events| Pipeline
  WF -->|poll| Ctrl
  Pipeline -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Pipeline
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PipelineEndpoint
  participant Q as PipelineQueue
  participant C as PipelineRequestConsumer
  participant W as PipelineWorkflow
  participant P as PlannerAgent
  participant E as Executor (Schema/SQL/Dataform/Validator)
  participant PE as PipelineEntity
  participant CTL as SystemControlEntity
  participant V as PipelineView

  U->>API: POST /api/pipelines {description}
  API->>Q: append PipelineSubmitted
  API-->>U: 202 {pipelineId}
  Q->>C: PipelineSubmitted
  C->>W: start({pipelineId, description})
  W->>PE: emit PipelineCreated (PLANNING)
  W->>P: PLAN_PIPELINE(description)
  P-->>W: PipelineLedger
  W->>PE: emit PipelinePlanned, status BUILDING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE_STEP(ledgers)
    P-->>W: Continue(BuildStepDecision)
    W->>W: DdlGuardrail.vet(decision)
    W->>E: runSingleTask(step)
    E-->>W: StepOutput
    W->>W: ValidationSuite.run(stepOutput)
    W->>PE: emit StepRecorded (StepEntry)
  end
  W->>P: COMPOSE_MANIFEST
  P-->>W: BuildManifest
  W->>PE: emit PipelineCompleted
  PE-->>V: project
  V-->>U: SSE update
```

## State machine — `PipelineEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> BUILDING: PipelinePlanned
  BUILDING --> BUILDING: StepRecorded / StepBlocked / LedgerRevised
  BUILDING --> COMPLETED: PipelineCompleted
  BUILDING --> FAILED: PipelineFailed
  BUILDING --> HALTED: PipelineHaltedOperator
  BUILDING --> STUCK: PipelineFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  PipelineEntity ||--o{ PipelinePlanned : emits
  PipelineEntity ||--o{ StepDispatched : emits
  PipelineEntity ||--o{ StepBlocked : emits
  PipelineEntity ||--o{ StepRecorded : emits
  PipelineEntity ||--o{ LedgerRevised : emits
  PipelineEntity ||--o{ PipelineCompleted : emits
  PipelineEntity ||--o{ PipelineFailed : emits
  PipelineEntity ||--o{ PipelineHaltedOperator : emits
  PipelineEntity ||--o{ PipelineFailedTimeout : emits
  PipelineView }o--|| PipelineEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  PipelineQueue ||--o{ PipelineSubmitted : emits
  PipelineRequestConsumer }o--|| PipelineQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `SchemaAnalystAgent` | `application/SchemaAnalystAgent.java` |
| `SqlComposerAgent` | `application/SqlComposerAgent.java` |
| `DataformModelerAgent` | `application/DataformModelerAgent.java` |
| `ValidatorAgent` | `application/ValidatorAgent.java` |
| `PipelineWorkflow` | `application/PipelineWorkflow.java` |
| `PipelineEntity` | `application/PipelineEntity.java` (state in `domain/Pipeline.java`, events in `domain/PipelineEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `PipelineQueue` | `application/PipelineQueue.java` |
| `PipelineView` | `application/PipelineView.java` |
| `PipelineRequestConsumer` | `application/PipelineRequestConsumer.java` |
| `PipelineSimulator` | `application/PipelineSimulator.java` |
| `StalePipelineMonitor` | `application/StalePipelineMonitor.java` |
| `DdlGuardrail` | `application/DdlGuardrail.java` |
| `ValidationSuite` | `application/ValidationSuite.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `PipelineEndpoint` | `api/PipelineEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any executor call, including a generous slack for a slow LLM), `ciGateStep` 30 s, `decideStep` 45 s, `completeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(PipelineWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same `(executor, step)` at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight step finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `PipelineEndpoint.submit` uses `(description, requestedBy)` over a 10 s window to dedupe `POST /api/pipelines`.
- **Stuck detection:** `StalePipelineMonitor` ticks every 30 s; `PipelineFailedTimeout` is non-fatal to other pipelines. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **CI gate determinism:** `ValidationSuite.run` is pure — it inspects only the `StepOutput` text and the fixture schema files loaded at startup. The same input always yields the same report, which keeps `StepEntry` events deterministic and replayable.
