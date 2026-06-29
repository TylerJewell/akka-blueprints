# PLAN — mle-pipeline

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
  Profiler[DataProfilerAgent]:::agent
  FeatEng[FeatureEngineerAgent]:::agent
  Trainer[ModelTrainerAgent]:::agent
  Evaluator[ModelEvaluatorAgent]:::agent

  WF[PipelineWorkflow]:::wf
  Run[PipelineRunEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RunQueue]:::ese
  View[PipelineRunView]:::view
  Consumer[RunRequestConsumer]:::cons
  Sim[RunSimulator]:::ta
  Stale[StaleRunMonitor]:::ta
  Drift[DriftWatchMonitor]:::ta
  API[PipelineEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_PIPELINE / DECIDE_STAGE / COMPOSE_REPORT| Planner
  WF -->|PROFILE_DATA| Profiler
  WF -->|ENGINEER_FEATURES| FeatEng
  WF -->|TRAIN_MODEL| Trainer
  WF -->|EVALUATE_MODEL| Evaluator
  WF -->|emit events| Run
  WF -->|poll| Ctrl
  Run -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 60s| Run
  Drift -.->|every 5min| Run
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PipelineEndpoint
  participant Q as RunQueue
  participant C as RunRequestConsumer
  participant W as PipelineWorkflow
  participant P as PlannerAgent
  participant S as Specialist (Profiler/FeatEng/Trainer/Evaluator)
  participant E as PipelineRunEntity
  participant CTL as SystemControlEntity
  participant V as PipelineRunView

  U->>API: POST /api/runs {datasetRef, objective}
  API->>Q: append RunSubmitted
  API-->>U: 202 {runId}
  Q->>C: RunSubmitted
  C->>W: start({runId, datasetRef, objective})
  W->>E: emit RunCreated (PLANNING)
  W->>P: PLAN_PIPELINE(objective)
  P-->>W: PipelineLedger
  W->>E: emit RunPlanned, status EXECUTING
  loop until Promote | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE_STAGE(ledgers)
    P-->>W: Continue(StageDispatch)
    W->>W: EvalGate.check(decision)
    W->>S: runSingleTask(stage)
    S-->>W: StageResult
    W->>E: emit StageRecorded (EvalEntry)
    W->>W: DriftEvaluator.check(entry, baseline)
  end
  W->>P: COMPOSE_REPORT
  P-->>W: ModelReport
  W->>E: emit RunCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `PipelineRunEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: RunPlanned
  EXECUTING --> EXECUTING: StageRecorded / StageGateFailed / LedgerRevised / DriftAlertRaised
  EXECUTING --> COMPLETED: RunCompleted
  EXECUTING --> GATE_FAILED: RunFailed (gate budget exhausted)
  EXECUTING --> FAILED: RunFailed
  EXECUTING --> HALTED: RunHaltedOperator / RunHaltedAutomatic
  EXECUTING --> STUCK: RunFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  GATE_FAILED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  PipelineRunEntity ||--o{ RunPlanned : emits
  PipelineRunEntity ||--o{ StageDispatched : emits
  PipelineRunEntity ||--o{ StageGateFailed : emits
  PipelineRunEntity ||--o{ StageRecorded : emits
  PipelineRunEntity ||--o{ LedgerRevised : emits
  PipelineRunEntity ||--o{ RunCompleted : emits
  PipelineRunEntity ||--o{ RunFailed : emits
  PipelineRunEntity ||--o{ RunHaltedAutomatic : emits
  PipelineRunEntity ||--o{ RunHaltedOperator : emits
  PipelineRunEntity ||--o{ RunFailedTimeout : emits
  PipelineRunEntity ||--o{ DriftAlertRaised : emits
  PipelineRunView }o--|| PipelineRunEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RunQueue ||--o{ RunSubmitted : emits
  RunRequestConsumer }o--|| RunQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `DataProfilerAgent` | `application/DataProfilerAgent.java` |
| `FeatureEngineerAgent` | `application/FeatureEngineerAgent.java` |
| `ModelTrainerAgent` | `application/ModelTrainerAgent.java` |
| `ModelEvaluatorAgent` | `application/ModelEvaluatorAgent.java` |
| `PipelineWorkflow` | `application/PipelineWorkflow.java` |
| `PipelineRunEntity` | `application/PipelineRunEntity.java` (state in `domain/PipelineRun.java`, events in `domain/RunEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RunQueue` | `application/RunQueue.java` |
| `PipelineRunView` | `application/PipelineRunView.java` |
| `RunRequestConsumer` | `application/RunRequestConsumer.java` |
| `RunSimulator` | `application/RunSimulator.java` |
| `StaleRunMonitor` | `application/StaleRunMonitor.java` |
| `DriftWatchMonitor` | `application/DriftWatchMonitor.java` |
| `EvalGate` | `application/EvalGate.java` |
| `DriftEvaluator` | `application/DriftEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `SpecialistTasks` | `application/SpecialistTasks.java` |
| `PipelineEndpoint` | `api/PipelineEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any specialist call), `decideStep` 45 s, `promoteStep` 60 s. Default recovery: `maxRetries(2).failoverTo(PipelineWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most three times in a row without a `Continue`; a fourth consecutive `Replan` is treated as `Fail`.
- **Gate budget:** the workflow tolerates at most two consecutive `StageGateFailed` records; a third triggers the Planner's `Fail` path.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight stage finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `PipelineEndpoint.submit` dedupes `POST /api/runs` by `(datasetRef, objective)` over a 10 s window.
- **Stuck detection:** `StaleRunMonitor` ticks every 60 s; tasks `EXECUTING` for > 10 minutes are marked `STUCK`. The workflow's `decideStep` checks entity status and exits if it reads `STUCK`.
- **Drift evaluation determinism:** `DriftEvaluator.check` is pure; the same `(MetricBundle, baseline)` pair always produces the same alerts, keeping `DriftAlertRaised` events deterministic and replayable.
