# PLAN — support-next-steps

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

  API[TicketEndpoint]:::ep
  Entity[TicketEntity]:::ese
  Sanitizer[TicketSanitizer]:::cons
  WF[TicketWorkflow]:::wf
  Agent[ResolutionAdvisorAgent]:::agent
  Scorer[RecommendationScorer]:::scorer
  View[TicketView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|TicketSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|adviseStep runSingleTask| Agent
  Agent -->|RecommendationSet| WF
  WF -->|recordRecommendation| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
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
  participant API as TicketEndpoint
  participant E as TicketEntity
  participant S as TicketSanitizer
  participant W as TicketWorkflow
  participant A as ResolutionAdvisorAgent
  participant Sc as RecommendationScorer

  U->>API: POST /api/tickets
  API->>E: submit(request)
  E-->>API: { ticketId }
  E-.->>S: TicketSubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(ticketId)
  W->>E: poll getTicket
  E-->>W: sanitized.isPresent()
  W->>E: markAdvising
  W->>A: runSingleTask(context + ticket.txt + resolution-library.txt)
  A-->>W: RecommendationSet
  W->>E: recordRecommendation(recommendation)
  W->>Sc: score(recommendation)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `TicketEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: TicketSanitized
  SANITIZED --> ADVISING: AdvisingStarted
  ADVISING --> RECOMMENDATION_RECORDED: RecommendationRecorded
  RECOMMENDATION_RECORDED --> EVALUATED: EvaluationScored
  ADVISING --> FAILED: TicketFailed (agent error)
  SUBMITTED --> FAILED: TicketFailed (sanitizer error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TicketEntity ||--o{ TicketSubmitted : emits
  TicketEntity ||--o{ TicketSanitized : emits
  TicketEntity ||--o{ AdvisingStarted : emits
  TicketEntity ||--o{ RecommendationRecorded : emits
  TicketEntity ||--o{ EvaluationScored : emits
  TicketEntity ||--o{ TicketFailed : emits
  TicketView }o--|| TicketEntity : projects
  TicketSanitizer }o--|| TicketEntity : subscribes
  TicketWorkflow }o--|| TicketEntity : reads-and-writes
  ResolutionAdvisorAgent ||--o{ RecommendationSet : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TicketEndpoint` | `api/TicketEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TicketEntity` | `application/TicketEntity.java` (state in `domain/Ticket.java`, events in `domain/TicketEvent.java`) |
| `TicketSanitizer` | `application/TicketSanitizer.java` |
| `TicketWorkflow` | `application/TicketWorkflow.java` |
| `ResolutionAdvisorAgent` | `application/ResolutionAdvisorAgent.java` (tasks in `application/TicketTasks.java`) |
| `RecommendationScorer` | `application/RecommendationScorer.java` |
| `TicketView` | `application/TicketView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `adviseStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TicketWorkflow::error)`. The 60 s on `adviseStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"ticket-" + ticketId` as the workflow id; the `TicketSanitizer` Consumer is allowed to redeliver `TicketSubmitted` events because `TicketEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized ticket is a no-op.
- **One agent per ticket**: the AutonomousAgent instance id is `"advisor-" + ticketId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps retries at 3.
- **Eval is synchronous and deterministic**: `RecommendationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same recommendation always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
