# PLAN — pr-reviewer

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

  Reviewer[ReviewerAgent]:::agent
  Alignment[AlignmentAgent]:::agent

  WF[ReviewWorkflow]:::wf
  ReviewE[ReviewEntity]:::ese
  Queue[PrQueue]:::ese
  View[ReviewsView]:::view
  Consumer[PrSubmissionConsumer]:::cons
  Sim[PrSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[ReviewEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit PR| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|review / revise| Reviewer
  WF -->|check alignment| Alignment
  WF -->|emit events| ReviewE
  ReviewE -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| ReviewE
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ReviewEndpoint
  participant Q as PrQueue
  participant C as PrSubmissionConsumer
  participant W as ReviewWorkflow
  participant R as ReviewerAgent
  participant A as AlignmentAgent
  participant E as ReviewEntity
  participant V as ReviewsView

  U->>API: POST /api/reviews {diffText, description}
  API->>Q: append PrSubmitted
  API-->>U: 202 {reviewId}
  Q->>C: PrSubmitted
  C->>W: start({reviewId, diffText, description, maxAttempts=4})
  W->>E: emit ReviewCreated (REVIEWING)

  W->>R: REVIEW(diffText, description)
  R-->>W: FeedbackDraft #1 (4 comments)
  W->>E: emit FeedbackDrafted (n=1)
  Note over W: guardrailStep (personal-critique scan)
  W->>E: emit FeedbackGuardrailVerdictRecorded (passed=true)
  W->>E: status CHECKING_ALIGNMENT
  W->>A: CHECK_ALIGNMENT(FeedbackDraft #1)
  A-->>W: AlignmentCheck{REVISE, score=3, 2 bullets}
  W->>E: emit FeedbackAlignmentChecked (n=1, REVISE)

  W->>R: REVISE_REVIEW(diffText, prior, notes)
  R-->>W: FeedbackDraft #2 (3 comments, refined)
  W->>E: emit FeedbackDrafted (n=2)
  W->>E: emit FeedbackGuardrailVerdictRecorded (passed=true)
  W->>A: CHECK_ALIGNMENT(FeedbackDraft #2)
  A-->>W: AlignmentCheck{APPROVE, score=5, rationale}
  W->>E: emit FeedbackAlignmentChecked (n=2, APPROVE)
  W->>E: emit ReviewApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ReviewEntity`

```mermaid
stateDiagram-v2
  [*] --> REVIEWING
  REVIEWING --> CHECKING_ALIGNMENT: FeedbackDrafted + guardrail passed
  REVIEWING --> REVIEWING: guardrail blocked, re-review
  CHECKING_ALIGNMENT --> REVIEWING: AlignmentCheck = REVISE, attempts < max
  CHECKING_ALIGNMENT --> APPROVED: AlignmentCheck = APPROVE
  CHECKING_ALIGNMENT --> REJECTED_FINAL: AlignmentCheck = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  ReviewEntity ||--o{ ReviewCreated : emits
  ReviewEntity ||--o{ FeedbackDrafted : emits
  ReviewEntity ||--o{ FeedbackGuardrailVerdictRecorded : emits
  ReviewEntity ||--o{ FeedbackAlignmentChecked : emits
  ReviewEntity ||--o{ ReviewApproved : emits
  ReviewEntity ||--o{ ReviewRejectedFinal : emits
  ReviewEntity ||--o{ EvalRecorded : emits
  ReviewsView }o--|| ReviewEntity : projects
  PrQueue ||--o{ PrSubmitted : emits
  PrSubmissionConsumer }o--|| PrQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `AlignmentAgent` | `application/AlignmentAgent.java` |
| `ReviewTasks` | `application/ReviewTasks.java` |
| `ReviewWorkflow` | `application/ReviewWorkflow.java` |
| `ReviewEntity` | `application/ReviewEntity.java` (state in `domain/Review.java`, events in `domain/ReviewEvent.java`) |
| `PrQueue` | `application/PrQueue.java` |
| `ReviewsView` | `application/ReviewsView.java` |
| `PrSubmissionConsumer` | `application/PrSubmissionConsumer.java` |
| `PrSimulator` | `application/PrSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `ReviewEndpoint` | `api/ReviewEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `reviewStep` and `alignmentStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `ReviewEndpoint.submit` deduplicates on `(diffText hash, submittedBy)` over a 10 s window.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(reviewId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `pr-reviewer.review.max-attempts` (default 4). The workflow checks the count BEFORE calling `reviewStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism ends the loop and preserves the best feedback and every alignment check on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it scans `FeedbackDraft.comments()` for personal language patterns (first/second-person singular references to the author) and either advances to `alignmentStep` or returns to `reviewStep` with a structured feedback note. The structured feedback never becomes an LLM-generated alignment note; it stays a deterministic `AlignmentNotes` payload with a single bullet.
