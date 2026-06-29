# PLAN — durable-workflow-recovery

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef helper fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[RecoveryEndpoint]:::ep
  Entity[ExecutionEntity]:::ese
  Consumer[CheckpointConsumer]:::cons
  WF[RecoveryWorkflow]:::wf
  Agent[WorkflowRecoveryAgent]:::agent
  Validator[ResponseValidator]:::helper
  Evaluator[HealthEvaluator]:::helper
  View[ExecutionView]:::view
  App[AppEndpoint]:::ep

  API -->|register| Entity
  Entity -.->|CheckpointRecorded| Consumer
  Consumer -->|markStalled| Entity
  Consumer -->|start workflow| WF
  WF -->|awaitStalledStep poll| Entity
  WF -->|analyzeStep runSingleTask| Agent
  Agent -->|RecoveryDecision| WF
  WF -->|validate| Validator
  Validator -->|pass/fail| WF
  WF -->|recordDecision| Entity
  WF -->|healthEvalStep score| Evaluator
  Evaluator -->|HealthScore| WF
  WF -->|recordHealth| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
  Consumer -->|periodic health tick| Evaluator
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as RecoveryEndpoint
  participant E as ExecutionEntity
  participant C as CheckpointConsumer
  participant W as RecoveryWorkflow
  participant A as WorkflowRecoveryAgent
  participant V as ResponseValidator
  participant Ev as HealthEvaluator

  U->>API: POST /api/executions
  API->>E: register(registration)
  E-->>API: { executionId }
  E-.->>C: CheckpointRecorded (×N)
  C->>C: check stall timeout
  C->>E: markStalled(snapshot)
  C->>W: start(executionId)
  W->>E: poll getExecution
  E-->>W: snapshot.isPresent()
  W->>E: startAnalysis
  W->>A: runSingleTask(metadata + attachment)
  A-->>W: RecoveryDecision
  W->>V: validate(decision, snapshot)
  V-->>W: valid
  W->>E: recordDecision(decision)
  W->>Ev: score(snapshot)
  Ev-->>W: HealthScore
  W->>E: recordHealth(health)
  E-.->>U: SSE event(HEALTH_SCORED)
```

## State machine — `ExecutionEntity`

```mermaid
stateDiagram-v2
  [*] --> REGISTERED
  REGISTERED --> RUNNING: CheckpointRecorded
  RUNNING --> RUNNING: CheckpointRecorded (progress)
  RUNNING --> STALLED: StallDetected
  STALLED --> ANALYZING: AnalysisStarted
  ANALYZING --> DECISION_RECORDED: DecisionRecorded (RESUME or ESCALATE)
  ANALYZING --> ABORTED: DecisionRecorded (ABORT)
  DECISION_RECORDED --> HEALTH_SCORED: HealthScored
  RUNNING --> FAILED: ExecutionFailed (system error)
  STALLED --> FAILED: ExecutionFailed (workflow error)
  HEALTH_SCORED --> [*]
  ABORTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ExecutionEntity ||--o{ ExecutionRegistered : emits
  ExecutionEntity ||--o{ CheckpointRecorded : emits
  ExecutionEntity ||--o{ StallDetected : emits
  ExecutionEntity ||--o{ AnalysisStarted : emits
  ExecutionEntity ||--o{ DecisionRecorded : emits
  ExecutionEntity ||--o{ HealthScored : emits
  ExecutionEntity ||--o{ ExecutionAborted : emits
  ExecutionEntity ||--o{ ExecutionFailed : emits
  ExecutionView }o--|| ExecutionEntity : projects
  CheckpointConsumer }o--|| ExecutionEntity : subscribes
  RecoveryWorkflow }o--|| ExecutionEntity : reads-and-writes
  WorkflowRecoveryAgent ||--o{ RecoveryDecision : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RecoveryEndpoint` | `api/RecoveryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ExecutionEntity` | `application/ExecutionEntity.java` (state in `domain/Execution.java`, events in `domain/ExecutionEvent.java`) |
| `CheckpointConsumer` | `application/CheckpointConsumer.java` |
| `RecoveryWorkflow` | `application/RecoveryWorkflow.java` |
| `WorkflowRecoveryAgent` | `application/WorkflowRecoveryAgent.java` (tasks in `application/RecoveryTasks.java`) |
| `ResponseValidator` | `application/ResponseValidator.java` |
| `HealthEvaluator` | `application/HealthEvaluator.java` |
| `ExecutionView` | `application/ExecutionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitStalledStep` 30 s, `analyzeStep` 60 s, `healthEvalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(RecoveryWorkflow::error)`. The 60 s on `analyzeStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"recovery-" + executionId` as the workflow id; the `CheckpointConsumer` is allowed to redeliver `CheckpointRecorded` events because `ExecutionEntity.markStalled` is event-version-guarded — a second stall attempt against an already-stalled execution is a no-op.
- **One agent per execution**: the AutonomousAgent instance id is `"recovery-" + executionId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps retry budget.
- **Validation failover**: when `ResponseValidator` rejects a candidate decision, `analyzeStep` fails over to the `error` step rather than silently discarding the result. The entity transitions to FAILED and the UI card shows the partial state.
- **Health eval is synchronous and deterministic**: `HealthEvaluator` runs in-process inside `healthEvalStep`. No LLM call, no external service — the same snapshot always scores the same. This is a deliberate single-agent guarantee.
- **Graceful degradation**: the halt mechanism means a RESUME verdict only dispatches a restart command after the agent has inspected the partial state. An ABORT verdict suppresses the restart entirely, preventing cascading failures on corrupted state.
- **Periodic health tick**: `CheckpointConsumer` runs a 60 s timer pass over all RUNNING and STALLED executions via `ExecutionView`. Each tick calls `HealthEvaluator.score(snapshot)` and emits `HealthScored` only if the score changed by more than 1 point since the previous evaluation.
