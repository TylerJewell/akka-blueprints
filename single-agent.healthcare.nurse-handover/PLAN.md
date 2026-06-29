# PLAN — nurse-handover

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
  classDef hitl fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[HandoverEndpoint]:::ep
  Entity[HandoverEntity]:::ese
  Sanitizer[ReportSanitizer]:::cons
  WF[HandoverWorkflow]:::wf
  Agent[HandoverAgent]:::agent
  View[HandoverView]:::view
  App[AppEndpoint]:::ep
  Clinician[Clinician Sign-off]:::hitl

  API -->|submit| Entity
  Entity -.->|ReportSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|summarizeStep runSingleTask| Agent
  Agent -->|HandoverSummary| WF
  WF -->|recordSummary| Entity
  WF -->|awaitSignoffStep poll| Entity
  Clinician -->|PATCH signoff| API
  API -->|requestSignoff| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant C as Clinician (UI)
  participant API as HandoverEndpoint
  participant E as HandoverEntity
  participant S as ReportSanitizer
  participant W as HandoverWorkflow
  participant A as HandoverAgent
  participant IC as Incoming Clinician

  C->>API: POST /api/handovers
  API->>E: submit(request)
  E-->>API: { handoverId }
  E-.->>S: ReportSubmitted
  S->>S: redact PHI
  S->>E: attachSanitized
  S->>W: start(handoverId)
  W->>E: poll getHandover
  E-->>W: sanitized.isPresent()
  W->>E: markSummarizing
  W->>A: runSingleTask(checklist + attachment)
  A-->>W: HandoverSummary
  W->>E: recordSummary(summary)
  W->>E: poll (awaitSignoffStep)
  IC->>API: PATCH /api/handovers/{id}/signoff
  API->>E: requestSignoff(signedOffBy)
  E-->>W: signoff.isPresent()
  W->>E: completeSignoff
  E-.->>C: SSE event(SIGNED_OFF)
```

## State machine — `HandoverEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: ReportSanitized
  SANITIZED --> SUMMARIZING: SummarizationStarted
  SUMMARIZING --> SUMMARY_READY: SummaryRecorded
  SUMMARY_READY --> AWAITING_SIGNOFF: SignoffRequested (auto)
  AWAITING_SIGNOFF --> SIGNED_OFF: HandoverSignedOff
  SUMMARIZING --> FAILED: HandoverFailed (agent error)
  SUBMITTED --> FAILED: HandoverFailed (sanitizer error)
  SIGNED_OFF --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  HandoverEntity ||--o{ ReportSubmitted : emits
  HandoverEntity ||--o{ ReportSanitized : emits
  HandoverEntity ||--o{ SummarizationStarted : emits
  HandoverEntity ||--o{ SummaryRecorded : emits
  HandoverEntity ||--o{ SignoffRequested : emits
  HandoverEntity ||--o{ HandoverSignedOff : emits
  HandoverEntity ||--o{ HandoverFailed : emits
  HandoverView }o--|| HandoverEntity : projects
  ReportSanitizer }o--|| HandoverEntity : subscribes
  HandoverWorkflow }o--|| HandoverEntity : reads-and-writes
  HandoverAgent ||--o{ HandoverSummary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `HandoverEndpoint` | `api/HandoverEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `HandoverEntity` | `application/HandoverEntity.java` (state in `domain/Handover.java`, events in `domain/HandoverEvent.java`) |
| `ReportSanitizer` | `application/ReportSanitizer.java` |
| `HandoverWorkflow` | `application/HandoverWorkflow.java` |
| `HandoverAgent` | `application/HandoverAgent.java` (tasks in `application/HandoverTasks.java`) |
| `HandoverView` | `application/HandoverView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `summarizeStep` 60 s, `awaitSignoffStep` 14400 s (4 h), `error` 5 s. Default step recovery `maxRetries(2).failoverTo(HandoverWorkflow::error)`. The 60 s on `summarizeStep` accommodates LLM latency (Lesson 4). The 4-hour window on `awaitSignoffStep` covers a full shift handover period.
- **Idempotency**: every workflow uses `"handover-" + handoverId` as the workflow id; the `ReportSanitizer` Consumer is allowed to redeliver `ReportSubmitted` events because `HandoverEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized handover is a no-op.
- **One agent per handover**: the AutonomousAgent instance id is `"handover-" + handoverId`. The agent's `capability(...).maxIterationsPerTask(3)` caps retries.
- **HITL gate is a poll, not an event push**: `awaitSignoffStep` polls the entity every 5 s and advances only when `handover.signoff().isPresent()`. The clinician's PATCH call lands on the entity and is detected on the next poll cycle. This keeps the workflow self-contained without requiring an external trigger mechanism.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
- **Single-agent invariant**: `HandoverAgent` is the only AutonomousAgent. The sign-off gate is a workflow polling loop — not a second LLM call.
