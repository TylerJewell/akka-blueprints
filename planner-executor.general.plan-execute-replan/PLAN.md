# PLAN — plan-execute-replan

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
  Replanner[ReplannerAgent]:::agent

  WF[ExecutionWorkflow]:::wf
  Goal[GoalEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[GoalQueue]:::ese
  View[GoalView]:::view
  Consumer[GoalRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stuck[StuckGoalMonitor]:::ta
  API[GoalEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|pause/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN| Planner
  WF -->|EXECUTE_STEP| Executor
  WF -->|REPLAN| Replanner
  WF -->|emit events| Goal
  WF -->|poll| Ctrl
  Goal -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Goal
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as GoalEndpoint
  participant Q as GoalQueue
  participant C as GoalRequestConsumer
  participant W as ExecutionWorkflow
  participant P as PlannerAgent
  participant E as ExecutorAgent
  participant R as ReplannerAgent
  participant G as GoalEntity
  participant CTL as SystemControlEntity
  participant V as GoalView

  U->>API: POST /api/goals {goal}
  API->>Q: append GoalSubmitted
  API-->>U: 202 {goalId}
  Q->>C: GoalSubmitted
  C->>W: start({goalId, goal})
  W->>G: emit GoalCreated (PLANNING)
  W->>P: PLAN(goal)
  P-->>W: ExecutionPlan
  W->>G: emit GoalPlanned, status EXECUTING
  loop until Conclude | Fail | Pause
    W->>CTL: get pause flag
    CTL-->>W: paused=false
    W->>W: ToolGuardrail.vet(toolAssignment)
    W->>E: EXECUTE_STEP(planStep)
    E-->>W: StepResult
    W->>G: emit StepObserved (Observation)
    W->>R: REPLAN(goal, plan, observations)
    R-->>W: ReplanDecision
    W->>W: ReplanQualityEvaluator.score(decision)
    W->>G: emit EvalRecorded
  end
  W->>G: emit GoalConcluded
  G-->>V: project
  V-->>U: SSE update
```

## State machine — `GoalEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: GoalPlanned
  EXECUTING --> EXECUTING: StepObserved / StepBlocked / PlanRevised / EvalRecorded
  EXECUTING --> CONCLUDED: GoalConcluded
  EXECUTING --> FAILED: GoalFailed
  EXECUTING --> PAUSED: GoalPaused
  EXECUTING --> STUCK: GoalFailedTimeout
  STUCK --> [*]
  CONCLUDED --> [*]
  FAILED --> [*]
  PAUSED --> [*]
```

## Entity model

```mermaid
erDiagram
  GoalEntity ||--o{ GoalPlanned : emits
  GoalEntity ||--o{ StepDispatched : emits
  GoalEntity ||--o{ StepBlocked : emits
  GoalEntity ||--o{ StepObserved : emits
  GoalEntity ||--o{ PlanRevised : emits
  GoalEntity ||--o{ EvalRecorded : emits
  GoalEntity ||--o{ GoalConcluded : emits
  GoalEntity ||--o{ GoalFailed : emits
  GoalEntity ||--o{ GoalPaused : emits
  GoalEntity ||--o{ GoalFailedTimeout : emits
  GoalView }o--|| GoalEntity : projects
  SystemControlEntity ||--o{ PauseRequested : emits
  SystemControlEntity ||--o{ PauseCleared : emits
  GoalQueue ||--o{ GoalSubmitted : emits
  GoalRequestConsumer }o--|| GoalQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `ExecutorAgent` | `application/ExecutorAgent.java` |
| `ReplannerAgent` | `application/ReplannerAgent.java` |
| `ExecutionWorkflow` | `application/ExecutionWorkflow.java` |
| `GoalEntity` | `application/GoalEntity.java` (state in `domain/Goal.java`, events in `domain/GoalEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `GoalQueue` | `application/GoalQueue.java` |
| `GoalView` | `application/GoalView.java` |
| `GoalRequestConsumer` | `application/GoalRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StuckGoalMonitor` | `application/StuckGoalMonitor.java` |
| `ToolGuardrail` | `application/ToolGuardrail.java` |
| `ReplanQualityEvaluator` | `application/ReplanQualityEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `ReplannerTasks` | `application/ReplannerTasks.java` |
| `GoalEndpoint` | `api/GoalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `executeStep` 90 s, `replanStep` 45 s, `evalStep` 30 s, `concludeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(ExecutionWorkflow::error)`.
- **Revision budget:** the Replanner may emit `Revise` at most three times; a fourth revision is treated as `Fail`.
- **Step failure budget:** each individual step may fail (STEP_FAILED or STEP_BLOCKED) at most twice before the Replanner is forced to `Revise` or `Fail`.
- **Pause poll:** every `checkPauseStep` reads `SystemControlEntity.get` synchronously — no caching. A pause arriving during an `executeStep` lets the in-flight step finish; the loop exits at the next `checkPauseStep`.
- **Idempotency:** `GoalEndpoint.submit` uses `(goal, requestedBy)` over a 10 s window to deduplicate `POST /api/goals`.
- **Stuck detection:** `StuckGoalMonitor` ticks every 30 s; goals `EXECUTING` for > 5 minutes are marked `STUCK`. The workflow's `decideStep` checks the entity's status and exits when it reads `STUCK`.
- **Evaluator determinism:** `ReplanQualityEvaluator.score` is pure — same inputs always produce the same score and rationale. The `EvalRecord` event is therefore deterministic and replayable.
