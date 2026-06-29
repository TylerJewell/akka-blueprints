# PLAN — docreview

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
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ReviewEndpoint]:::ep
  Entity[ReviewEntity]:::ese
  Sanitizer[DocumentSanitizer]:::cons
  WF[ReviewWorkflow]:::wf
  Agent[DocumentReviewerAgent]:::agent
  Guard[VerdictGuardrail]:::guard
  Scorer[EvaluationScorer]:::guard
  View[ReviewView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|DocumentSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|reviewStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|ReviewVerdict| WF
  WF -->|recordVerdict| Entity
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
  participant API as ReviewEndpoint
  participant E as ReviewEntity
  participant S as DocumentSanitizer
  participant W as ReviewWorkflow
  participant A as DocumentReviewerAgent
  participant G as VerdictGuardrail
  participant Sc as EvaluationScorer

  U->>API: POST /api/reviews
  API->>E: submit(request)
  E-->>API: { reviewId }
  E-.->>S: DocumentSubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(reviewId)
  W->>E: poll getReview
  E-->>W: sanitized.isPresent()
  W->>E: markReviewing
  W->>A: runSingleTask(instructions + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: ReviewVerdict
  W->>E: recordVerdict(verdict)
  W->>Sc: score(verdict, instructions)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `ReviewEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: DocumentSanitized
  SANITIZED --> REVIEWING: ReviewStarted
  REVIEWING --> VERDICT_RECORDED: VerdictRecorded
  VERDICT_RECORDED --> EVALUATED: EvaluationScored
  REVIEWING --> FAILED: ReviewFailed (agent error)
  SUBMITTED --> FAILED: ReviewFailed (sanitizer error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ReviewEntity ||--o{ DocumentSubmitted : emits
  ReviewEntity ||--o{ DocumentSanitized : emits
  ReviewEntity ||--o{ ReviewStarted : emits
  ReviewEntity ||--o{ VerdictRecorded : emits
  ReviewEntity ||--o{ EvaluationScored : emits
  ReviewEntity ||--o{ ReviewFailed : emits
  ReviewView }o--|| ReviewEntity : projects
  DocumentSanitizer }o--|| ReviewEntity : subscribes
  ReviewWorkflow }o--|| ReviewEntity : reads-and-writes
  DocumentReviewerAgent ||--o{ ReviewVerdict : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ReviewEndpoint` | `api/ReviewEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ReviewEntity` | `application/ReviewEntity.java` (state in `domain/Review.java`, events in `domain/ReviewEvent.java`) |
| `DocumentSanitizer` | `application/DocumentSanitizer.java` |
| `ReviewWorkflow` | `application/ReviewWorkflow.java` |
| `DocumentReviewerAgent` | `application/DocumentReviewerAgent.java` (tasks in `application/ReviewTasks.java`) |
| `VerdictGuardrail` | `application/VerdictGuardrail.java` |
| `EvaluationScorer` | `application/EvaluationScorer.java` |
| `ReviewView` | `application/ReviewView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `reviewStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ReviewWorkflow::error)`. The 60 s on `reviewStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"review-" + reviewId` as the workflow id; the `DocumentSanitizer` Consumer is allowed to redeliver `DocumentSubmitted` events because `ReviewEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized review is a no-op.
- **One agent per review**: the AutonomousAgent instance id is `"reviewer-" + reviewId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `VerdictGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `reviewStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `EvaluationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same verdict always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
