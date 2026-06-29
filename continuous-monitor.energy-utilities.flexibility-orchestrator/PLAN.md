# PLAN — flexibility-orchestrator

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

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

  Poller[GridSignalPoller]:::ta
  Signal[DemandSignalEntity]:::ese
  Evaluator[DispatchEvaluatorAgent]:::agent
  WF[DispatchWorkflow]:::wf
  Dispatch[DispatchEntity]:::ese
  Program[FlexibilityProgramEntity]:::ese
  Pricing[PricingConsumer]:::cons
  View[DispatchView]:::view
  EvalRunner[EvalRunner]:::ta
  API[FlexibilityEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 60s| Signal
  Signal -.->|ELEVATED/CRITICAL| WF
  WF -->|call| Evaluator
  WF -->|budget check| Program
  WF -->|emit events| Dispatch
  WF -->|debit budget| Program
  Dispatch -.->|DispatchApproved| Pricing
  Pricing -->|applyPricingAdjustment| Program
  Dispatch -.->|projects| View
  API -->|approve/cancel| Dispatch
  API -->|query/SSE| View
  API -->|query| Program
  EvalRunner -.->|every 30m| Dispatch
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as GridSignalPoller
  participant S as DemandSignalEntity
  participant W as DispatchWorkflow
  participant E as DispatchEvaluatorAgent
  participant D as DispatchEntity
  participant FP as FlexibilityProgramEntity
  participant U as Operator (UI)
  participant API as FlexibilityEndpoint

  P->>S: emit DemandSignalReceived (ELEVATED)
  S->>W: start({signalId, signal})
  W->>D: emit DispatchRequested
  W->>E: evaluate(signal, program)
  E-->>W: DispatchRecommendation (curtailmentMW=35)
  W->>D: emit DispatchEvaluated
  W->>FP: getProgram (budgetCheck)
  FP-->>W: budgetRemaining=80 MW (OK)
  W->>D: emit DispatchAwaitingApproval
  Note over D,U: Workflow pauses — curtailment > threshold
  U->>API: POST /api/dispatches/{id}/approve
  API->>D: emit DispatchApproved
  D-->>W: resume
  W->>D: emit DispatchExecuting
  W->>FP: debitBudget(35 MW)
  W->>D: emit DispatchCompleted
```

## State machine — `DispatchEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> EVALUATING: DispatchEvaluated
  EVALUATING --> BUDGET_EXCEEDED: budget cap hit
  EVALUATING --> AWAITING_APPROVAL: curtailment > threshold
  EVALUATING --> EXECUTING: curtailment <= threshold
  AWAITING_APPROVAL --> APPROVED: operator Approve
  AWAITING_APPROVAL --> CANCELLED: operator Cancel
  APPROVED --> EXECUTING
  EXECUTING --> COMPLETED: DispatchCompleted
  BUDGET_EXCEEDED --> [*]
  CANCELLED --> [*]
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  DemandSignalEntity ||--o{ DemandSignalReceived : emits
  DispatchEntity ||--o{ DispatchRequested : emits
  DispatchEntity ||--o{ DispatchEvaluated : emits
  DispatchEntity ||--o{ DispatchAwaitingApproval : emits
  DispatchEntity ||--o{ DispatchApproved : emits
  DispatchEntity ||--o{ DispatchCancelled : emits
  DispatchEntity ||--o{ DispatchExecuting : emits
  DispatchEntity ||--o{ DispatchCompleted : emits
  DispatchEntity ||--o{ BudgetCapExceeded : emits
  DispatchEntity ||--o{ EvalScored : emits
  FlexibilityProgramEntity ||--o{ BudgetDebited : emits
  FlexibilityProgramEntity ||--o{ PricingAdjustmentApplied : emits
  DispatchView }o--|| DispatchEntity : projects
  PricingConsumer }o--|| DispatchEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GridSignalPoller` | `application/GridSignalPoller.java` |
| `DemandSignalEntity` | `application/DemandSignalEntity.java` |
| `DispatchEvaluatorAgent` | `application/DispatchEvaluatorAgent.java` |
| `DispatchWorkflow` | `application/DispatchWorkflow.java` |
| `DispatchEntity` | `application/DispatchEntity.java` (state in `domain/DispatchRecord.java`, events in `domain/DispatchEvent.java`) |
| `FlexibilityProgramEntity` | `application/FlexibilityProgramEntity.java` (state in `domain/FlexibilityProgram.java`) |
| `PricingConsumer` | `application/PricingConsumer.java` |
| `DispatchView` | `application/DispatchView.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `FlexibilityEndpoint` | `api/FlexibilityEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Budget check ordering**: the budget check runs inside `DispatchWorkflow.budgetCheckStep` — a synchronous read of `FlexibilityProgramEntity` state — before any HITL pause. If the budget is exhausted, the workflow ends immediately without prompting the operator.
- **Per-step timeout**: evaluateStep 30 s. On timeout, the workflow emits `BudgetCapExceeded` (conservative default: if we cannot evaluate, do not dispatch).
- **HITL gate**: `DispatchWorkflow` pauses in AWAITING_APPROVAL using the poll-the-entity idiom; every 5 s it checks if `decision.isPresent()`. No auto-timeout — dispatches wait indefinitely for operator action.
- **Idempotency**: every workflow uses `requestId` (derived from `signalId`) as the workflow id so duplicate signal events fold into one workflow instance.
- **Eval sampling**: per tick, EvalRunner picks up to 5 COMPLETED dispatches with no `evalScore`, oldest-first.
