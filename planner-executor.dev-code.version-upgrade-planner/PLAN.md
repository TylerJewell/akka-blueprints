# PLAN — version-upgrade-planner

Architectural sketch consumed by `/akka:plan`. Diagrams render on the generated system's Architecture tab.

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

  Planner[UpgradePlannerAgent]:::agent
  Compat[CompatibilityCheckerAgent]:::agent
  Tests[TestRunnerAgent]:::agent
  Migr[MigrationApplierAgent]:::agent

  WF[UpgradeWorkflow]:::wf
  Job[UpgradeJobEntity]:::ese
  Appr[ApprovalEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[UpgradeRequestQueue]:::ese
  View[UpgradeJobView]:::view
  Consumer[UpgradeRequestConsumer]:::cons
  Sim[UpgradeRequestSimulator]:::ta
  Stuck[StuckJobMonitor]:::ta
  API[UpgradeJobEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  API -->|approve/reject| Appr
  Sim -.->|every 120s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_UPGRADE / DECIDE_PHASE / COMPOSE_REPORT| Planner
  WF -->|CHECK_COMPAT| Compat
  WF -->|RUN_TESTS| Tests
  WF -->|APPLY_MIGRATION| Migr
  WF -->|emit events| Job
  WF -->|write/poll| Appr
  WF -->|poll| Ctrl
  Job -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 60s| Job
```

## Interaction sequence — J1 (happy path with approval gate)

```mermaid
sequenceDiagram
  autonumber
  participant U as User / Reviewer
  participant API as UpgradeJobEndpoint
  participant Q as UpgradeRequestQueue
  participant C as UpgradeRequestConsumer
  participant W as UpgradeWorkflow
  participant P as UpgradePlannerAgent
  participant E as Executor (Compat/Test/Migr)
  participant J as UpgradeJobEntity
  participant A as ApprovalEntity
  participant CTL as SystemControlEntity
  participant V as UpgradeJobView

  U->>API: POST /api/jobs {sourceVersion, targetVersion}
  API->>Q: append UpgradeSubmitted
  API-->>U: 202 {jobId}
  Q->>C: UpgradeSubmitted
  C->>W: start({jobId, sourceVersion, targetVersion})
  W->>J: emit JobCreated (PLANNING)
  W->>P: PLAN_UPGRADE(sourceVersion, targetVersion)
  P-->>W: PlanLedger (phases)
  W->>J: emit JobPlanned, status EXECUTING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE_PHASE(ledgers)
    P-->>W: Continue(PhaseDecision{requiresApproval=true})
    W->>A: createApproval(request)
    W->>J: emit ApprovalRequested, status AWAITING_APPROVAL
    W->>W: poll ApprovalEntity until resolved
    U->>API: POST /api/approvals/{id}/approve
    API->>A: resolve(approved=true)
    A-->>W: ApprovalResolved(approved=true)
    W->>J: emit ApprovalGranted, status EXECUTING
    W->>E: runSingleTask(phase)
    E-->>W: PhaseResult
    W->>W: ciGateStep (if MIGRATION phase)
    W->>J: emit PhaseRecorded (ProgressEntry)
  end
  W->>P: COMPOSE_REPORT
  P-->>W: UpgradeReport
  W->>J: emit JobCompleted
  J-->>V: project
  V-->>U: SSE update
```

## State machine — `UpgradeJobEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: JobPlanned
  EXECUTING --> AWAITING_APPROVAL: ApprovalRequested
  AWAITING_APPROVAL --> EXECUTING: ApprovalGranted
  AWAITING_APPROVAL --> EXECUTING: ApprovalRejected (PhaseBlocked, replan)
  EXECUTING --> EXECUTING: PhaseRecorded / PhaseBlocked / CiGateFailed / LedgerRevised
  EXECUTING --> COMPLETED: JobCompleted
  EXECUTING --> FAILED: JobFailed
  EXECUTING --> HALTED: JobHaltedOperator
  EXECUTING --> STUCK: JobFailedTimeout
  AWAITING_APPROVAL --> STUCK: JobFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  UpgradeJobEntity ||--o{ JobPlanned : emits
  UpgradeJobEntity ||--o{ PhaseDispatched : emits
  UpgradeJobEntity ||--o{ ApprovalRequested : emits
  UpgradeJobEntity ||--o{ ApprovalGranted : emits
  UpgradeJobEntity ||--o{ ApprovalRejected : emits
  UpgradeJobEntity ||--o{ PhaseBlocked : emits
  UpgradeJobEntity ||--o{ PhaseRecorded : emits
  UpgradeJobEntity ||--o{ CiGateFailed : emits
  UpgradeJobEntity ||--o{ LedgerRevised : emits
  UpgradeJobEntity ||--o{ JobCompleted : emits
  UpgradeJobEntity ||--o{ JobFailed : emits
  UpgradeJobEntity ||--o{ JobHaltedOperator : emits
  UpgradeJobEntity ||--o{ JobFailedTimeout : emits
  ApprovalEntity ||--o{ ApprovalCreated : emits
  ApprovalEntity ||--o{ ApprovalResolved : emits
  UpgradeJobView }o--|| UpgradeJobEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  UpgradeRequestQueue ||--o{ UpgradeSubmitted : emits
  UpgradeRequestConsumer }o--|| UpgradeRequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `UpgradePlannerAgent` | `application/UpgradePlannerAgent.java` |
| `CompatibilityCheckerAgent` | `application/CompatibilityCheckerAgent.java` |
| `TestRunnerAgent` | `application/TestRunnerAgent.java` |
| `MigrationApplierAgent` | `application/MigrationApplierAgent.java` |
| `UpgradeWorkflow` | `application/UpgradeWorkflow.java` |
| `UpgradeJobEntity` | `application/UpgradeJobEntity.java` (state in `domain/UpgradeJob.java`, events in `domain/UpgradeJobEvent.java`) |
| `ApprovalEntity` | `application/ApprovalEntity.java` |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `UpgradeRequestQueue` | `application/UpgradeRequestQueue.java` |
| `UpgradeJobView` | `application/UpgradeJobView.java` |
| `UpgradeRequestConsumer` | `application/UpgradeRequestConsumer.java` |
| `UpgradeRequestSimulator` | `application/UpgradeRequestSimulator.java` |
| `StuckJobMonitor` | `application/StuckJobMonitor.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `UpgradeJobEndpoint` | `api/UpgradeJobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 90 s, `plannerDecideStep` 60 s, `approvalGateStep` 3600 s (human response window), `dispatchStep` 180 s, `ciGateStep` 120 s, `composeReportStep` 60 s.
- **Approval gate:** the `approvalGateStep` polls `ApprovalEntity.get` in a bounded retry loop with back-off. The 3600 s step timeout is the outer bound; the `StuckJobMonitor` catches jobs that exceed 10 minutes of combined wall-clock `EXECUTING`+`AWAITING_APPROVAL` time.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** at most three consecutive attempts on the same `(executor, phaseId)` pair; a fourth becomes `Fail`.
- **CI gate:** non-blocking on first failure — the gate appends `CiGateFailed` and returns to the planner rather than terminating immediately. The planner may add a rollback phase.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously; an operator halt mid-phase lets the phase finish.
- **Stuck detection:** `StuckJobMonitor` ticks every 60 s; `AWAITING_APPROVAL` jobs count toward the 10-minute threshold.
