# PLAN — seo-rubric-critic

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

  Rubric[RubricAgent]:::agent
  Optimizer[OptimizerAgent]:::agent

  WF[AuditWorkflow]:::wf
  Article[ArticleEntity]:::ese
  Queue[SubmissionQueue]:::ese
  View[ArticlesView]:::view
  Consumer[SubmissionConsumer]:::cons
  Sim[ArticleSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[AuditEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue article| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|score article| Rubric
  WF -->|revise article| Optimizer
  WF -->|emit events| Article
  Article -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Article
```

## Interaction sequence — J1 (convergence on round 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as AuditEndpoint
  participant Q as SubmissionQueue
  participant C as SubmissionConsumer
  participant W as AuditWorkflow
  participant R as RubricAgent
  participant O as OptimizerAgent
  participant A as ArticleEntity
  participant V as ArticlesView

  U->>API: POST /api/articles {title, bodyText, keyword}
  API->>Q: append ArticleSubmitted
  API-->>U: 202 {articleId}
  Q->>C: ArticleSubmitted
  C->>W: start({articleId, title, wordCountCeiling, maxRounds=4})
  W->>A: emit ArticleCreated (SCORING)

  W->>R: SCORE_ARTICLE(draft, keyword)
  R-->>W: RubricScore{FAIL, score=54, 4 failing dimensions}
  W->>A: emit RoundScoreRecorded (n=1, FAIL)
  W->>A: status REVISING
  Note over W: guardrailStep (deterministic word-count check)
  W->>O: REVISE_ARTICLE(draft, rubricFeedback)
  O-->>W: ArticleDraft (1100 words)
  W->>A: emit RoundDraftRecorded (n=2)
  W->>A: emit RoundGuardrailVerdictRecorded (passed=true)

  W->>R: SCORE_ARTICLE(revised draft, keyword)
  R-->>W: RubricScore{PASS, score=82}
  W->>A: emit RoundScoreRecorded (n=2, PASS)
  W->>A: emit ArticleApproved (n=2)
  A-->>V: project
  V-->>U: SSE update
```

## State machine — `ArticleEntity`

```mermaid
stateDiagram-v2
  [*] --> SCORING
  SCORING --> REVISING: RubricScore = FAIL, rounds < max
  SCORING --> APPROVED: RubricScore = PASS
  SCORING --> FAILED_FINAL: RubricScore = FAIL, rounds = max
  REVISING --> REVISING: guardrail blocked, re-revise
  REVISING --> SCORING: guardrail passed, re-score
  APPROVED --> [*]
  FAILED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  ArticleEntity ||--o{ ArticleCreated : emits
  ArticleEntity ||--o{ RoundDraftRecorded : emits
  ArticleEntity ||--o{ RoundGuardrailVerdictRecorded : emits
  ArticleEntity ||--o{ RoundScoreRecorded : emits
  ArticleEntity ||--o{ ArticleApproved : emits
  ArticleEntity ||--o{ ArticleFailedFinal : emits
  ArticleEntity ||--o{ EvalRecorded : emits
  ArticlesView }o--|| ArticleEntity : projects
  SubmissionQueue ||--o{ ArticleSubmitted : emits
  SubmissionConsumer }o--|| SubmissionQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RubricAgent` | `application/RubricAgent.java` |
| `OptimizerAgent` | `application/OptimizerAgent.java` |
| `AuditTasks` | `application/AuditTasks.java` |
| `AuditWorkflow` | `application/AuditWorkflow.java` |
| `ArticleEntity` | `application/ArticleEntity.java` (state in `domain/Article.java`, events in `domain/ArticleEvent.java`) |
| `SubmissionQueue` | `application/SubmissionQueue.java` |
| `ArticlesView` | `application/ArticlesView.java` |
| `SubmissionConsumer` | `application/SubmissionConsumer.java` |
| `ArticleSimulator` | `application/ArticleSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `AuditEndpoint` | `api/AuditEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `scoreStep` and `reviseStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — the workflow degrades to `FAILED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `AuditEndpoint.submit` uses `(title, submittedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(articleId, roundNumber)` so a tick that fires twice for the same round is a no-op on the entity side.
- **maxRounds ceiling:** read from `seo-rubric-critic.audit.max-rounds` (default 4). The workflow checks the count BEFORE calling `reviseStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only terminal path; it preserves the highest-scoring draft and every rubric score on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it counts the words in the draft and either advances to `scoreStep` or returns to `reviseStep` with a structured feedback note. The structured feedback never becomes an LLM-generated critique; it stays a deterministic `RubricFeedback` payload with a single bullet.
