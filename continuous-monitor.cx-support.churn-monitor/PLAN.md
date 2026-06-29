# PLAN — churn-monitor

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

  Poller[AccountPoller]:::ta
  Queue[AccountSnapshotQueue]:::ese
  Sanitizer[PiiSanitizer]:::cons
  Scorer[ChurnScoringAgent]:::agent
  Advisor[RetentionAdvisorAgent]:::agent
  WF[ChurnWorkflow]:::wf
  Entity[AccountChurnEntity]:::ese
  View[ChurnView]:::view
  EvalRunner[DriftFairnessEvalRunner]:::ta
  API[ChurnEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 60s| Queue
  Queue -.->|subscribes| Sanitizer
  Sanitizer -->|emit SnapshotSanitized| Entity
  Entity -.->|on sanitized| WF
  WF -->|call| Scorer
  WF -->|call if HIGH| Advisor
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|action/dismiss| Entity
  API -->|query/SSE| View
  EvalRunner -.->|every 6h| Entity
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as AccountPoller
  participant Q as AccountSnapshotQueue
  participant S as PiiSanitizer
  participant E as AccountChurnEntity
  participant W as ChurnWorkflow
  participant SC as ChurnScoringAgent
  participant A as RetentionAdvisorAgent
  participant U as User (UI)
  participant API as ChurnEndpoint

  P->>Q: emit SnapshotReceived
  Q->>S: SnapshotReceived
  S->>E: emit SnapshotSanitized
  E->>W: start({accountId, sanitized})
  W->>SC: score(sanitized)
  SC-->>W: HIGH risk ChurnScore
  W->>E: emit AccountScored
  W->>A: advise(sanitized, score)
  A-->>W: RetentionPlan
  W->>E: emit RetentionAdvised (status AWAITING_ACTION)
  Note over E,U: Workflow pauses — only HIGH risk accounts reach here
  U->>API: POST /api/churn/{accountId}/action
  API->>E: emit ActionRecorded
  E-->>W: resume → end
```

## State machine — `AccountChurnEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: SnapshotSanitized
  SANITIZED --> SCORED: AccountScored
  SCORED --> LOW_RISK_CLOSED: riskLevel = LOW
  SCORED --> MEDIUM_RISK_CLOSED: riskLevel = MEDIUM
  SCORED --> ADVISED: riskLevel = HIGH
  ADVISED --> AWAITING_ACTION
  AWAITING_ACTION --> ACTIONED: manager Mark Actioned
  AWAITING_ACTION --> DISMISSED: manager Dismiss
  LOW_RISK_CLOSED --> [*]
  MEDIUM_RISK_CLOSED --> [*]
  ACTIONED --> [*]
  DISMISSED --> [*]
```

## Entity model

```mermaid
erDiagram
  AccountChurnEntity ||--o{ SnapshotReceived : emits
  AccountChurnEntity ||--o{ SnapshotSanitized : emits
  AccountChurnEntity ||--o{ AccountScored : emits
  AccountChurnEntity ||--o{ RetentionAdvised : emits
  AccountChurnEntity ||--o{ ActionRecorded : emits
  AccountChurnEntity ||--o{ AccountDismissed : emits
  AccountChurnEntity ||--o{ AccountClosed : emits
  AccountChurnEntity ||--o{ EvalCompleted : emits
  ChurnView }o--|| AccountChurnEntity : projects
  AccountSnapshotQueue ||--o{ SnapshotReceived : emits
  PiiSanitizer }o--|| AccountSnapshotQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AccountPoller` | `application/AccountPoller.java` |
| `AccountSnapshotQueue` | `application/AccountSnapshotQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ChurnScoringAgent` | `application/ChurnScoringAgent.java` |
| `RetentionAdvisorAgent` | `application/RetentionAdvisorAgent.java` |
| `ChurnWorkflow` | `application/ChurnWorkflow.java` |
| `AccountChurnEntity` | `application/AccountChurnEntity.java` (state in `domain/AccountChurnState.java`, events in `domain/AccountChurnEvent.java`) |
| `ChurnView` | `application/ChurnView.java` |
| `DriftFairnessEvalRunner` | `application/DriftFairnessEvalRunner.java` |
| `ChurnEndpoint` | `api/ChurnEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: scorer 15 s, advisor 30 s. On timeout, treat as HIGH and escalate to AWAITING_ACTION with an error note in the retention plan.
- **HITL gate**: `ChurnWorkflow` pauses in AWAITING_ACTION only for HIGH risk accounts, using the workflow's poll-the-entity idiom; on each poll, if `decision.isPresent()` it advances.
- **Auto-close**: LOW and MEDIUM risk accounts emit `AccountClosed` from within the workflow without entering AWAITING_ACTION — no human attention required.
- **Idempotency**: every workflow uses `accountId` as the workflow id so duplicate sanitize events fold into one workflow.
- **Eval batching**: per tick, DriftFairnessEvalRunner picks up to 50 accounts scored since the last eval run, oldest-first, sends them as a single batch to DriftFairnessEvalJudge.
