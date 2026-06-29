# PLAN — brand-presentation-builder

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Builder[BuilderAgent]:::agent
  Reviewer[BrandReviewerAgent]:::agent

  WF[PresentationWorkflow]:::wf
  Pres[PresentationEntity]:::ese
  Queue[RequestQueue]:::ese
  View[PresentationsView]:::view
  Consumer[PresentationRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[PresentationEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue brief| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|build / revise| Builder
  WF -->|review| Reviewer
  WF -->|emit events| Pres
  Pres -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Pres
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PresentationEndpoint
  participant Q as RequestQueue
  participant C as PresentationRequestConsumer
  participant W as PresentationWorkflow
  participant B as BuilderAgent
  participant R as BrandReviewerAgent
  participant E as PresentationEntity
  participant V as PresentationsView

  U->>API: POST /api/presentations {topic, targetAudience, slideCount}
  API->>Q: append BriefSubmitted
  API-->>U: 202 {presentationId}
  Q->>C: BriefSubmitted
  C->>W: start({presentationId, topic, targetAudience, slideCount, wordsPerSlide=80, maxAttempts=4})
  W->>E: emit PresentationCreated (BUILDING)

  W->>B: BUILD(topic, targetAudience, slideCount, wordsPerSlide)
  B-->>W: SlideSet #1 (5 slides, all ≤80 words)
  W->>E: emit AttemptBuilt (n=1)
  Note over W: guardrailStep (deterministic word-count check per slide)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>E: status REVIEWING
  W->>R: REVIEW(SlideSet #1, brief)
  R-->>W: BrandReview{REVISE, score=3, 3 bullets}
  W->>E: emit AttemptBrandReviewed (n=1, REVISE)

  W->>B: REVISE_BUILD(topic, priorSlideSet, brandFeedback)
  B-->>W: SlideSet #2 (5 slides, revised)
  W->>E: emit AttemptBuilt (n=2)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>R: REVIEW(SlideSet #2, brief)
  R-->>W: BrandReview{APPROVE, score=5, rationale}
  W->>E: emit AttemptBrandReviewed (n=2, APPROVE)
  W->>E: emit PresentationApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `PresentationEntity`

```mermaid
stateDiagram-v2
  [*] --> BUILDING
  BUILDING --> REVIEWING: AttemptBuilt + guardrail passed
  BUILDING --> BUILDING: guardrail blocked, re-build
  REVIEWING --> BUILDING: BrandReview = REVISE, attempts < max
  REVIEWING --> APPROVED: BrandReview = APPROVE
  REVIEWING --> REJECTED_FINAL: BrandReview = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  PresentationEntity ||--o{ PresentationCreated : emits
  PresentationEntity ||--o{ AttemptBuilt : emits
  PresentationEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  PresentationEntity ||--o{ AttemptBrandReviewed : emits
  PresentationEntity ||--o{ PresentationApproved : emits
  PresentationEntity ||--o{ PresentationRejectedFinal : emits
  PresentationEntity ||--o{ BrandEvalRecorded : emits
  PresentationsView }o--|| PresentationEntity : projects
  RequestQueue ||--o{ BriefSubmitted : emits
  PresentationRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BuilderAgent` | `application/BuilderAgent.java` |
| `BrandReviewerAgent` | `application/BrandReviewerAgent.java` |
| `PresentationTasks` | `application/PresentationTasks.java` |
| `PresentationWorkflow` | `application/PresentationWorkflow.java` |
| `PresentationEntity` | `application/PresentationEntity.java` (state in `domain/Presentation.java`, events in `domain/PresentationEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `PresentationsView` | `application/PresentationsView.java` |
| `PresentationRequestConsumer` | `application/PresentationRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `PresentationEndpoint` | `api/PresentationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `buildStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `PresentationEndpoint.submit` uses `(topic, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordBrandEval` calls on `(presentationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `brand-presentation.refinement.max-attempts` (default 4). The workflow checks the count BEFORE calling `buildStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best slide set and every round of feedback on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it checks each `slide.wordCount() <= wordsPerSlide` and either advances to `reviewStep` or returns to `buildStep` with a structured feedback note listing the offending slide numbers. The structured feedback never becomes an LLM-generated review; it stays a deterministic `BrandFeedback` payload.
