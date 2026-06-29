# PLAN — planner-executor-general-planner-executor

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
  Executor[ExecutorAgent]:::agent

  WF[PlanWorkflow]:::wf
  PlanE[PlanEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[GoalQueue]:::ese
  View[PlanView]:::view
  Consumer[GoalRequestConsumer]:::cons
  Sim[GoalSimulator]:::ta
  Stuck[StuckPlanMonitor]:::ta
  API[PlanEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|CREATE_PLAN / DECIDE_STEP / COMPOSE_OUTCOME| Planner
  WF -->|EXECUTE_STEP| Executor
  WF -->|emit events| PlanE
  WF -->|poll| Ctrl
  PlanE -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| PlanE
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PlanEndpoint
  participant Q as GoalQueue
  participant C as GoalRequestConsumer
  participant W as PlanWorkflow
  participant P as PlannerAgent
  participant E as ExecutorAgent
  participant PE as PlanEntity
  participant CTL as SystemControlEntity
  participant V as PlanView

  U->>API: POST /api/plans {goal}
  API->>Q: append GoalSubmitted
  API-->>U: 202 {planId}
  Q->>C: GoalSubmitted
  C->>W: start({planId, goal})
  W->>PE: emit PlanCreated (PLANNING)
  W->>P: CREATE_PLAN(goal)
  P-->>W: PlanLedger
  W->>PE: emit PlanStarted, status EXECUTING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE_STEP(ledger, stepLog)
    P-->>W: Continue(StepDecision)
    W->>W: guardrail.vet(decision)
    W->>E: EXECUTE_STEP(step)
    E-->>W: StepResult
    W->>PE: emit StepRecorded (StepEntry)
    W->>W: StepSafetyEvaluator.evaluate(entry)
  end
  W->>P: COMPOSE_OUTCOME
  P-->>W: PlanOutcome
  W->>PE: emit PlanCompleted
  PE-->>V: project
  V-->>U: SSE update
```

## State machine — `PlanEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: PlanStarted
  EXECUTING --> EXECUTING: StepRecorded / StepBlocked / PlanRevised
  EXECUTING --> COMPLETED: PlanCompleted
  EXECUTING --> FAILED: PlanFailed
  EXECUTING --> HALTED: PlanHaltedOperator / PlanHaltedAutomatic
  EXECUTING --> STUCK: PlanFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  PlanEntity ||--o{ PlanCreated : emits
  PlanEntity ||--o{ PlanStarted : emits
  PlanEntity ||--o{ StepProposed : emits
  PlanEntity ||--o{ StepBlocked : emits
  PlanEntity ||--o{ StepRecorded : emits
  PlanEntity ||--o{ PlanRevised : emits
  PlanEntity ||--o{ PlanCompleted : emits
  PlanEntity ||--o{ PlanFailed : emits
  PlanEntity ||--o{ PlanHaltedAutomatic : emits
  PlanEntity ||--o{ PlanHaltedOperator : emits
  PlanEntity ||--o{ PlanFailedTimeout : emits
  PlanView }o--|| PlanEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  GoalQueue ||--o{ GoalSubmitted : emits
  GoalRequestConsumer }o--|| GoalQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `ExecutorAgent` | `application/ExecutorAgent.java` |
| `PlanWorkflow` | `application/PlanWorkflow.java` |
| `PlanEntity` | `application/PlanEntity.java` (state in `domain/Plan.java`, events in `domain/PlanEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `GoalQueue` | `application/GoalQueue.java` |
| `PlanView` | `application/PlanView.java` |
| `GoalRequestConsumer` | `application/GoalRequestConsumer.java` |
| `GoalSimulator` | `application/GoalSimulator.java` |
| `StuckPlanMonitor` | `application/StuckPlanMonitor.java` |
| `StepGuardrail` | `application/StepGuardrail.java` |
| `StepSafetyEvaluator` | `application/StepSafetyEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `PlanEndpoint` | `api/PlanEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `createPlanStep` 60 s, `proposeStep` 45 s, `executeStep` 120 s, `decideStep` 45 s, `composeOutcomeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(PlanWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same step text at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during `executeStep` lets the in-flight step finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `PlanEndpoint.submit` uses `(goal, requestedBy)` over a 10 s window to deduplicate `POST /api/plans`.
- **Stuck detection:** `StuckPlanMonitor` ticks every 30 s; plans `EXECUTING` for > 5 minutes are marked `STUCK`.
- **Guardrail determinism:** `StepGuardrail.vet` is pure; the same `StepDecision` always yields the same verdict, keeping events deterministic and replayable.
