# PLAN — itsm-change-planner

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
  TestRunner[TestRunnerAgent]:::agent
  Backout[BackoutAgent]:::agent

  WF[ChangeWorkflow]:::wf
  Change[ChangeEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[ChangeQueue]:::ese
  View[ChangeView]:::view
  Consumer[ChangeRequestConsumer]:::cons
  Sim[ChangeSimulator]:::ta
  Stuck[StuckChangeMonitor]:::ta
  API[ChangeEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|approve/reject| Change
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_CHANGE / REVISE_STEP| Planner
  WF -->|EXECUTE_STEP| Executor
  WF -->|RUN_TEST| TestRunner
  WF -->|EXECUTE_BACKOUT| Backout
  WF -->|emit events| Change
  WF -->|poll cab status| Change
  WF -->|poll| Ctrl
  Change -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 60s| Change
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ChangeEndpoint
  participant Q as ChangeQueue
  participant C as ChangeRequestConsumer
  participant W as ChangeWorkflow
  participant P as PlannerAgent
  participant E as ExecutorAgent
  participant T as TestRunnerAgent
  participant CE as ChangeEntity
  participant CAB as CAB Reviewer
  participant CTL as SystemControlEntity
  participant V as ChangeView

  U->>API: POST /api/changes {summary, ciName}
  API->>Q: append ChangeSubmitted
  API-->>U: 202 {changeId}
  Q->>C: ChangeSubmitted
  C->>W: start({changeId, summary})
  W->>CE: emit ChangeCreated (PLANNING)
  W->>P: PLAN_CHANGE(request + history fixtures)
  P-->>W: ChangeLedger
  W->>CE: emit ChangePlanned (AWAITING_CAB)
  loop CAB poll
    W->>CE: getChange
    CE-->>W: status=AWAITING_CAB
  end
  CAB->>API: POST /api/changes/{id}/approve
  API->>CE: emit ChangeApproved (APPROVED→EXECUTING)
  loop per implementation step
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>W: guardrail.vet(nextStep)
    W->>E: EXECUTE_STEP(step)
    E-->>W: StepResult
    W->>T: RUN_TEST(testStep)
    T-->>W: TestResult passed=true
    W->>CE: emit StepExecuted + TestPassed
  end
  W->>CE: emit ChangeImplemented (IMPLEMENTED)
  CE-->>V: project
  V-->>U: SSE update
```

## State machine — `ChangeEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> AWAITING_CAB: ChangePlanned
  AWAITING_CAB --> APPROVED: ChangeApproved
  AWAITING_CAB --> REJECTED: ChangeRejected
  APPROVED --> EXECUTING: execution loop starts
  EXECUTING --> EXECUTING: StepExecuted / TestPassed / StepBlocked
  EXECUTING --> ROLLING_BACK: TestFailed
  EXECUTING --> IMPLEMENTED: ChangeImplemented
  EXECUTING --> FAILED: ChangeFailed
  EXECUTING --> HALTED: ChangeHaltedOperator
  EXECUTING --> STUCK: ChangeFailedTimeout
  ROLLING_BACK --> ROLLED_BACK: ChangeRolledBack
  ROLLING_BACK --> STUCK: ChangeFailedTimeout
  REJECTED --> [*]
  IMPLEMENTED --> [*]
  ROLLED_BACK --> [*]
  FAILED --> [*]
  HALTED --> [*]
  STUCK --> [*]
```

## Entity model

```mermaid
erDiagram
  ChangeEntity ||--o{ ChangePlanned : emits
  ChangeEntity ||--o{ ChangeApproved : emits
  ChangeEntity ||--o{ ChangeRejected : emits
  ChangeEntity ||--o{ StepBlocked : emits
  ChangeEntity ||--o{ StepExecuted : emits
  ChangeEntity ||--o{ TestPassed : emits
  ChangeEntity ||--o{ TestFailed : emits
  ChangeEntity ||--o{ BackoutStepExecuted : emits
  ChangeEntity ||--o{ ChangeImplemented : emits
  ChangeEntity ||--o{ ChangeRolledBack : emits
  ChangeEntity ||--o{ ChangeFailed : emits
  ChangeEntity ||--o{ ChangeHaltedOperator : emits
  ChangeEntity ||--o{ ChangeFailedTimeout : emits
  ChangeView }o--|| ChangeEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  ChangeQueue ||--o{ ChangeSubmitted : emits
  ChangeRequestConsumer }o--|| ChangeQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `ExecutorAgent` | `application/ExecutorAgent.java` |
| `TestRunnerAgent` | `application/TestRunnerAgent.java` |
| `BackoutAgent` | `application/BackoutAgent.java` |
| `ChangeWorkflow` | `application/ChangeWorkflow.java` |
| `ChangeEntity` | `application/ChangeEntity.java` (state in `domain/Change.java`, events in `domain/ChangeEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `ChangeQueue` | `application/ChangeQueue.java` |
| `ChangeView` | `application/ChangeView.java` |
| `ChangeRequestConsumer` | `application/ChangeRequestConsumer.java` |
| `ChangeSimulator` | `application/ChangeSimulator.java` |
| `StuckChangeMonitor` | `application/StuckChangeMonitor.java` |
| `ProductionTouchGuardrail` | `application/ProductionTouchGuardrail.java` |
| `BackoutEvaluator` | `application/BackoutEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `ChangeEndpoint` | `api/ChangeEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 90 s, `executeStepStep` 120 s (executor agent call), `testStepStep` 120 s (test runner call), `backoutStep` 120 s per backout step. `cabApprovalPollStep` uses a short sleep-and-poll loop with no agent call; retry on entity read errors.
- **Replan budget:** the planner may be asked to revise a blocked step at most twice per step; a third rejection of the same step transitions the change to `FAILED`.
- **CAB timeout:** changes sitting in `AWAITING_CAB` for more than 48 hours are marked `STUCK` by `StuckChangeMonitor`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during an `executeStepStep` lets the in-flight step pair finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `ChangeEndpoint.submit` uses `(summary, ciName, requestedBy)` over a 30 s window to deduplicate `POST /api/changes`.
- **Backout ordering:** `BackoutAgent` processes backout steps in descending sequence order (reverse of implementation) so partial rollbacks undo work in the correct order.
- **Stuck detection:** `StuckChangeMonitor` ticks every 60 s; `ChangeFailedTimeout` is non-fatal to other changes. The workflow reads entity status at the top of each loop iteration and exits on `STUCK`.
