# PLAN — data-processing-planner

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
  Executor[JobExecutorAgent]:::agent

  WF[PipelineWorkflow]:::wf
  Pipeline[PipelineEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[PipelineQueue]:::ese
  View[PipelineView]:::view
  Consumer[PipelineRequestConsumer]:::cons
  Sim[PipelineSimulator]:::ta
  Stuck[StuckPipelineMonitor]:::ta
  API[PipelineEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE_OUTPUT| Planner
  WF -->|EXECUTE_STEP| Executor
  WF -->|emit events| Pipeline
  WF -->|poll| Ctrl
  Pipeline -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Pipeline
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
  participant E as JobExecutorAgent
  participant PE as PipelineEntity
  participant CTL as SystemControlEntity
  participant V as PipelineView

  U->>API: POST /api/pipelines {description, targetDataset}
  API->>Q: append PipelineSubmitted
  API-->>U: 202 {runId}
  Q->>C: PipelineSubmitted
  C->>W: start({runId, description})
  W->>PE: emit PipelineRunCreated (PLANNING)
  W->>P: PLAN(description)
  P-->>W: JobLedger
  W->>PE: emit PipelineRunPlanned, status RUNNING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE(ledgers)
    P-->>W: ContinueDispatch(JobDispatch)
    W->>W: CostGuardrail.vet(dispatch)
    W->>E: EXECUTE_STEP(dispatch)
    E-->>W: JobStepResult
    W->>W: SecretScrubber.scrub(output)
    W->>PE: emit JobStepRecorded (RunEntry)
  end
  W->>P: COMPOSE_OUTPUT
  P-->>W: PipelineOutput
  W->>PE: emit PipelineRunCompleted
  PE-->>V: project
  V-->>U: SSE update
```

## State machine — `PipelineEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> RUNNING: PipelineRunPlanned
  RUNNING --> RUNNING: JobStepRecorded / JobStepBlocked / JobLedgerRevised
  RUNNING --> COMPLETED: PipelineRunCompleted
  RUNNING --> FAILED: PipelineRunFailed
  RUNNING --> HALTED: PipelineRunHaltedOperator
  RUNNING --> STUCK: PipelineRunFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  PipelineEntity ||--o{ PipelineRunPlanned : emits
  PipelineEntity ||--o{ JobStepDispatched : emits
  PipelineEntity ||--o{ JobStepBlocked : emits
  PipelineEntity ||--o{ JobStepRecorded : emits
  PipelineEntity ||--o{ JobLedgerRevised : emits
  PipelineEntity ||--o{ PipelineRunCompleted : emits
  PipelineEntity ||--o{ PipelineRunFailed : emits
  PipelineEntity ||--o{ PipelineRunHaltedOperator : emits
  PipelineEntity ||--o{ PipelineRunFailedTimeout : emits
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
| `JobExecutorAgent` | `application/JobExecutorAgent.java` |
| `PipelineWorkflow` | `application/PipelineWorkflow.java` |
| `PipelineEntity` | `application/PipelineEntity.java` (state in `domain/PipelineRun.java`, events in `domain/PipelineEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `PipelineQueue` | `application/PipelineQueue.java` |
| `PipelineView` | `application/PipelineView.java` |
| `PipelineRequestConsumer` | `application/PipelineRequestConsumer.java` |
| `PipelineSimulator` | `application/PipelineSimulator.java` |
| `StuckPipelineMonitor` | `application/StuckPipelineMonitor.java` |
| `CostGuardrail` | `application/CostGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `PipelineEndpoint` | `api/PipelineEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `executeStep` 180 s (covers job simulation plus generous slack for slow LLM), `decideStep` 45 s, `composeOutputStep` 60 s. Default recovery: `maxRetries(2).failoverTo(PipelineWorkflow::error)`.
- **Replan budget:** the planner may emit `RevisePlan` at most twice in a row without a `ContinueDispatch` in between; a third consecutive `RevisePlan` is treated as `Fail`.
- **Failure budget:** the planner may emit `ContinueDispatch` on the same `(engine, stepSpec)` at most two times; a third attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during an `executeStep` lets the in-flight step finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `PipelineEndpoint.submit` uses `(description, targetDataset, requestedBy)` over a 10 s window to dedupe `POST /api/pipelines`.
- **Stuck detection:** `StuckPipelineMonitor` ticks every 30 s; runs `RUNNING` for > 8 minutes are marked `STUCK`. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; the same input always yields the same scrubbed output, keeping `RunEntry` events deterministic and replayable.
