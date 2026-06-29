# PLAN — lambda-error-diagnoser

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

  Poller[LogPoller]:::ta
  Queue[LogQueue]:::ese
  Normalizer[LogNormalizer]:::cons
  Diagnoser[DiagnosisAgent]:::agent
  Judge[EvalJudge]:::agent
  WF[IncidentWorkflow]:::wf
  Entity[IncidentEntity]:::ese
  View[IncidentView]:::view
  API[IncidentEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 20s| Queue
  Queue -.->|subscribes| Normalizer
  Normalizer -->|emit ErrorNormalised| Entity
  Entity -.->|on normalised| WF
  WF -->|call| Diagnoser
  WF -->|call immediately after| Judge
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|dismiss/reopen| Entity
  API -->|query/SSE| View
```

## Interaction sequence — J1

```mermaid
sequenceDiagram
  autonumber
  participant P as LogPoller
  participant Q as LogQueue
  participant N as LogNormalizer
  participant E as IncidentEntity
  participant W as IncidentWorkflow
  participant D as DiagnosisAgent
  participant J as EvalJudge
  participant U as User (UI)
  participant API as IncidentEndpoint

  P->>Q: emit ErrorDetected
  Q->>N: ErrorDetected
  N->>E: emit ErrorNormalised
  E->>W: start({incidentId, normalised})
  W->>D: diagnose(normalised)
  D-->>W: DiagnosisResult
  W->>E: emit DiagnosisProduced
  W->>J: eval(normalised, diagnosis)
  J-->>W: EvalResult
  W->>E: emit DiagnosisEvaluated
  W->>E: emit IncidentResolved (status RESOLVED)
  Note over E,U: Incident visible with diagnosis + score
  U->>API: POST /api/incidents/{id}/dismiss
  API->>E: emit IncidentDismissed
```

## State machine — `IncidentEntity`

```mermaid
stateDiagram-v2
  [*] --> DETECTED
  DETECTED --> NORMALISED: ErrorNormalised
  NORMALISED --> DIAGNOSED: DiagnosisProduced
  DIAGNOSED --> EVALUATED: DiagnosisEvaluated
  EVALUATED --> RESOLVED: IncidentResolved
  RESOLVED --> DISMISSED: operator Dismiss
  DISMISSED --> REOPENED: operator Re-open
  REOPENED --> DISMISSED: operator Dismiss
  RESOLVED --> [*]
  DISMISSED --> [*]
```

## Entity model

```mermaid
erDiagram
  IncidentEntity ||--o{ ErrorDetected : emits
  IncidentEntity ||--o{ ErrorNormalised : emits
  IncidentEntity ||--o{ DiagnosisProduced : emits
  IncidentEntity ||--o{ DiagnosisEvaluated : emits
  IncidentEntity ||--o{ IncidentResolved : emits
  IncidentEntity ||--o{ IncidentDismissed : emits
  IncidentEntity ||--o{ IncidentReopened : emits
  IncidentView }o--|| IncidentEntity : projects
  LogQueue ||--o{ ErrorDetected : emits
  LogNormalizer }o--|| LogQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `LogPoller` | `application/LogPoller.java` |
| `LogQueue` | `application/LogQueue.java` |
| `LogNormalizer` | `application/LogNormalizer.java` |
| `DiagnosisAgent` | `application/DiagnosisAgent.java` |
| `EvalJudge` | `application/EvalJudge.java` |
| `IncidentWorkflow` | `application/IncidentWorkflow.java` |
| `IncidentEntity` | `application/IncidentEntity.java` (state in `domain/Incident.java`, events in `domain/IncidentEvent.java`) |
| `IncidentView` | `application/IncidentView.java` |
| `IncidentEndpoint` | `api/IncidentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: diagnose step 15 s; eval step 10 s. On diagnose timeout, emit IncidentResolved with a low-confidence stub and score=1 so the incident does not stall indefinitely.
- **On-incident eval**: the eval call happens in the same workflow execution, in the step immediately after diagnosis. There is no separate scheduler or sampling logic — every incident gets scored.
- **Idempotency**: every workflow uses `incidentId` as the workflow id, so duplicate normalise events fold into one workflow.
- **Severity ordering in the UI**: the view sorts CRITICAL and HIGH incidents to the top of the list regardless of arrival time; the underlying getAllIncidents query returns all rows and the client sorts.
