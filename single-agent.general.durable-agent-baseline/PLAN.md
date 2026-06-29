# PLAN — durable-agent-baseline

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
  classDef scorer fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[WorkOrderEndpoint]:::ep
  Entity[WorkOrderEntity]:::ese
  Monitor[RuntimeMonitor]:::cons
  WF[WorkOrderWorkflow]:::wf
  Agent[WorkOrderAgent]:::agent
  Evaluator[PerformanceEvaluator]:::scorer
  View[WorkOrderView]:::view
  App[AppEndpoint]:::ep

  API -->|initiate + start| Entity
  API -->|start workflow| WF
  Entity -.->|StepStarted / StepCompleted| Monitor
  Monitor -->|recordAlert| Entity
  WF -->|initStep markRunning| Entity
  WF -->|agentStep runSingleTask| Agent
  Agent -->|WorkOrderResult| WF
  WF -->|recordResult| Entity
  WF -->|evalStep score| Evaluator
  Evaluator -->|EvalScore| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as WorkOrderEndpoint
  participant E as WorkOrderEntity
  participant WF as WorkOrderWorkflow
  participant Mon as RuntimeMonitor
  participant A as WorkOrderAgent
  participant Ev as PerformanceEvaluator

  U->>API: POST /api/work-orders
  API->>E: initiate(order)
  E-->>API: { workOrderId }
  API->>WF: start(workOrderId)
  WF->>E: markRunning
  WF->>A: runSingleTask(steps instructions)
  E-.->>Mon: StepStarted (each step)
  Mon->>Mon: open stall window
  A-->>WF: WorkOrderResult
  E-.->>Mon: StepCompleted (each step)
  Mon->>Mon: close stall window
  WF->>E: recordResult(result)
  WF->>Ev: score(workOrderRun)
  Ev-->>WF: EvalScore
  WF->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `WorkOrderEntity`

```mermaid
stateDiagram-v2
  [*] --> INITIATED
  INITIATED --> RUNNING: WorkOrderStarted
  RUNNING --> STEP_COMPLETED: StepCompleted
  STEP_COMPLETED --> RUNNING: next step starts
  RUNNING --> STALLED: AlertRecorded (STALL)
  STALLED --> RUNNING: agent resumes
  RUNNING --> COMPLETED: WorkOrderCompleted
  COMPLETED --> EVALUATED: PerformanceEvaluated
  RUNNING --> FAILED: WorkOrderFailed (error)
  STALLED --> FAILED: WorkOrderFailed (exhausted)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  WorkOrderEntity ||--o{ WorkOrderInitiated : emits
  WorkOrderEntity ||--o{ WorkOrderStarted : emits
  WorkOrderEntity ||--o{ StepStarted : emits
  WorkOrderEntity ||--o{ StepCompleted : emits
  WorkOrderEntity ||--o{ StepFailed : emits
  WorkOrderEntity ||--o{ WorkOrderCompleted : emits
  WorkOrderEntity ||--o{ WorkOrderFailed : emits
  WorkOrderEntity ||--o{ AlertRecorded : emits
  WorkOrderEntity ||--o{ PerformanceEvaluated : emits
  WorkOrderView }o--|| WorkOrderEntity : projects
  RuntimeMonitor }o--|| WorkOrderEntity : subscribes-and-writes
  WorkOrderWorkflow }o--|| WorkOrderEntity : reads-and-writes
  WorkOrderAgent ||--o{ WorkOrderResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `WorkOrderEndpoint` | `api/WorkOrderEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `WorkOrderEntity` | `application/WorkOrderEntity.java` (state in `domain/WorkOrderRun.java`, events in `domain/WorkOrderEvent.java`) |
| `RuntimeMonitor` | `application/RuntimeMonitor.java` |
| `WorkOrderWorkflow` | `application/WorkOrderWorkflow.java` |
| `WorkOrderAgent` | `application/WorkOrderAgent.java` (tasks in `application/WorkOrderTasks.java`) |
| `PerformanceEvaluator` | `application/PerformanceEvaluator.java` |
| `WorkOrderView` | `application/WorkOrderView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `initStep` 10 s, `agentStep` 120 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(WorkOrderWorkflow::error)`. The 120 s on `agentStep` accommodates multi-step LLM execution across potentially 5 iterations (Lesson 4).
- **Durability**: the Akka Workflow journals its step position; a JVM restart leaves `WorkOrderWorkflow` at its last committed step boundary. The `WorkOrderEntity`'s event log is the ground truth for step outcomes.
- **Idempotency**: every workflow uses `"wf-" + workOrderId` as the workflow id; re-submitting the same `workOrderId` returns the existing entity state rather than creating a duplicate.
- **One agent per work order**: the AutonomousAgent instance id is `"agent-" + workOrderId`, giving each run its own conversation context. `maxIterationsPerTask(5)` accommodates step-level retries within a single task.
- **Monitor restart recovery**: `RuntimeMonitor` re-reads the entity's open step windows on restart before committing to a stall check, so a JVM bounce during a long step does not suppress a legitimate stall alert.
- **Eval is synchronous and deterministic**: `PerformanceEvaluator` runs in-process inside `evalStep`. No LLM call — the same run always scores the same. This is the single-agent invariant.
- **No saga / no compensation**: every step is append-only event writes or a single-task agent call. Nothing external requires rollback.
