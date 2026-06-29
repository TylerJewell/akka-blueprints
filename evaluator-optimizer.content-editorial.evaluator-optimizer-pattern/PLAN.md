# PLAN — evaluator-optimizer-workflow

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

  Writer[WriterAgent]:::agent
  Reviewer[ReviewerAgent]:::agent

  WF[HeadlineWorkflow]:::wf
  HL[HeadlineEntity]:::ese
  Queue[SubmissionQueue]:::ese
  View[HeadlinesView]:::view
  Consumer[SubmissionConsumer]:::cons
  Sim[BriefSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[EditorialEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue article| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|draft / revise| Writer
  WF -->|evaluate| Reviewer
  WF -->|emit events| HL
  HL -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| HL
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EditorialEndpoint
  participant Q as SubmissionQueue
  participant C as SubmissionConsumer
  participant W as HeadlineWorkflow
  participant WA as WriterAgent
  participant RA as ReviewerAgent
  participant E as HeadlineEntity
  participant V as HeadlinesView

  U->>API: POST /api/headlines {summary, ceiling}
  API->>Q: append ArticleSubmitted
  API-->>U: 202 {headlineId}
  Q->>C: ArticleSubmitted
  C->>W: start({headlineId, summary, ceiling, maxAttempts=4})
  W->>E: emit HeadlineCreated (DRAFTING)

  W->>WA: DRAFT(summary, ceiling)
  WA-->>W: HeadlineDraft #1 (11 words)
  W->>E: emit AttemptDrafted (n=1)
  Note over W: guardrailStep (deterministic word count check)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>E: status REVIEWING
  W->>RA: EVALUATE(HeadlineDraft #1)
  RA-->>W: Review{REVISE, score=3, 3 bullets}
  W->>E: emit AttemptReviewed (n=1, REVISE)

  W->>WA: REVISE_DRAFT(summary, prior, notes)
  WA-->>W: HeadlineDraft #2 (9 words)
  W->>E: emit AttemptDrafted (n=2)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>RA: EVALUATE(HeadlineDraft #2)
  RA-->>W: Review{APPROVE, score=5, rationale}
  W->>E: emit AttemptReviewed (n=2, APPROVE)
  W->>E: emit HeadlineApproved (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `HeadlineEntity`

```mermaid
stateDiagram-v2
  [*] --> DRAFTING
  DRAFTING --> REVIEWING: AttemptDrafted + guardrail passed
  DRAFTING --> DRAFTING: guardrail blocked, re-draft
  REVIEWING --> DRAFTING: Review = REVISE, attempts < max
  REVIEWING --> APPROVED: Review = APPROVE
  REVIEWING --> REJECTED_FINAL: Review = REVISE, attempts = max
  APPROVED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  HeadlineEntity ||--o{ HeadlineCreated : emits
  HeadlineEntity ||--o{ AttemptDrafted : emits
  HeadlineEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  HeadlineEntity ||--o{ AttemptReviewed : emits
  HeadlineEntity ||--o{ HeadlineApproved : emits
  HeadlineEntity ||--o{ HeadlineRejectedFinal : emits
  HeadlineEntity ||--o{ EvalRecorded : emits
  HeadlinesView }o--|| HeadlineEntity : projects
  SubmissionQueue ||--o{ ArticleSubmitted : emits
  SubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `WriterAgent` | `application/WriterAgent.java` |
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `EditorialTasks` | `application/EditorialTasks.java` |
| `HeadlineWorkflow` | `application/HeadlineWorkflow.java` |
| `HeadlineEntity` | `application/HeadlineEntity.java` (state in `domain/Headline.java`, events in `domain/HeadlineEvent.java`) |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `HeadlinesView` | `application/HeadlinesView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `BriefSimulator` | `application/BriefSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `EditorialEndpoint` | `api/EditorialEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `draftStep` and `reviewStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `EditorialEndpoint.submit` uses `(summary, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(headlineId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `evaluator-optimizer-workflow.headline.max-attempts` (default 4). The workflow checks the count BEFORE calling `draftStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt-at-ceiling path is the only terminal safety valve; it preserves the best-scoring draft and every review on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it counts the words in the draft and either advances to `reviewStep` or returns to `draftStep` with a structured feedback note. The structured feedback never becomes an LLM-generated review; it stays a deterministic `ReviewNotes` payload with a single bullet.
