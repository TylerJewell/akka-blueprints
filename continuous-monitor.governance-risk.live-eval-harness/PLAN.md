# PLAN — live-eval-harness

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

  Poller[DecisionPoller]:::ta
  Queue[DecisionQueue]:::ese
  Sanitizer[DecisionSanitizer]:::cons
  Eval[RubricEvalAgent]:::agent
  Drift[DriftWatchAgent]:::agent
  WF[EvalOrchestrationWorkflow]:::wf
  Entity[DecisionEntity]:::ese
  DriftEntity[DriftSnapshotEntity]:::ese
  View[EvalView]:::view
  DriftSampler[DriftSampler]:::ta
  API[EvalEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 10s| Queue
  Queue -.->|subscribes| Sanitizer
  Sanitizer -->|emit DecisionSanitized| Entity
  Entity -.->|on sanitized| WF
  WF -->|call| Eval
  WF -->|emit EvalScored/Alarm| Entity
  Entity -.->|projects| View
  DriftEntity -.->|projects| View
  API -->|query/SSE| View
  DriftSampler -.->|every 15m| View
  DriftSampler -->|call| Drift
  DriftSampler -->|emit DriftAssessed| DriftEntity
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as DecisionPoller
  participant Q as DecisionQueue
  participant S as DecisionSanitizer
  participant E as DecisionEntity
  participant W as EvalOrchestrationWorkflow
  participant R as RubricEvalAgent
  participant UI as User (UI)
  participant API as EvalEndpoint

  P->>Q: emit DecisionReceived
  Q->>S: DecisionReceived
  S->>E: registerDecision + attachSanitized
  S->>W: start({decisionId, sanitized})
  W->>R: score(sanitized, rubricDefinition)
  R-->>W: EvalResult{overallScore=2}
  W->>E: emit EvalScored
  W->>E: emit EvalAlarm (score <= threshold)
  E-->>View: project EvalViewRow (status=ALARMED)
  UI->>API: GET /api/evals/sse
  API-->>UI: event: eval-update {decisionId, status=ALARMED, overallScore=2}
```

## State machine — `DecisionEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: DecisionSanitized
  SANITIZED --> EVALUATED: EvalScored
  EVALUATED --> OK: score above threshold
  EVALUATED --> ALARMED: score at or below threshold
  OK --> [*]
  ALARMED --> [*]
```

## Entity model

```mermaid
erDiagram
  DecisionEntity ||--o{ DecisionReceived : emits
  DecisionEntity ||--o{ DecisionSanitized : emits
  DecisionEntity ||--o{ EvalScored : emits
  DecisionEntity ||--o{ EvalAlarm : emits
  DecisionEntity ||--o{ DecisionOk : emits
  EvalView }o--|| DecisionEntity : projects
  EvalView }o--|| DriftSnapshotEntity : projects
  DecisionQueue ||--o{ DecisionReceived : emits
  DecisionSanitizer }o--|| DecisionQueue : subscribes
  DriftSnapshotEntity ||--o{ DriftAssessed : emits
  DriftSnapshotEntity ||--o{ DriftAlarm : emits
  DriftSnapshotEntity ||--o{ DriftResolved : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DecisionPoller` | `application/DecisionPoller.java` |
| `DecisionQueue` | `application/DecisionQueue.java` |
| `DecisionSanitizer` | `application/DecisionSanitizer.java` |
| `RubricEvalAgent` | `application/RubricEvalAgent.java` |
| `DriftWatchAgent` | `application/DriftWatchAgent.java` |
| `EvalOrchestrationWorkflow` | `application/EvalOrchestrationWorkflow.java` |
| `DecisionEntity` | `application/DecisionEntity.java` (state in `domain/DecisionRecord.java`, events in `domain/DecisionEvent.java`) |
| `DriftSnapshotEntity` | `application/DriftSnapshotEntity.java` |
| `DriftSampler` | `application/DriftSampler.java` |
| `EvalView` | `application/EvalView.java` |
| `EvalEndpoint` | `api/EvalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: eval step 20 s. On timeout, emit `EvalScored` with `overallScore=1` and `summary="eval-timeout"` so the decision does not stall.
- **Alarm threshold**: configurable; default 3. Decisions with `overallScore <= threshold` emit `EvalAlarm`.
- **Idempotency**: every workflow uses `decisionId` as the workflow id so duplicate sanitize events fold into one workflow instance.
- **Drift sampling**: per tick, `DriftSampler` takes the 50 most recent EVALUATED/OK/ALARMED decisions. On an empty window (service just started), it emits `DriftAssessed` with `status=OK` and narrative "Insufficient data".
- **DriftSnapshotEntity** is a singleton (id="global"). All sessions share the single drift state.
