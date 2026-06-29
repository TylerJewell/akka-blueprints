# PLAN — self-critiquing-writer-loop

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
  Critic[CriticAgent]:::agent

  WF[RefinementWorkflow]:::wf
  Article[ArticleEntity]:::ese
  Queue[RequestQueue]:::ese
  View[ArticlesView]:::view
  Consumer[DraftRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[WritingEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue brief| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|draft / revise| Writer
  WF -->|evaluate| Critic
  WF -->|emit events| Article
  Article -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Article
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as WritingEndpoint
  participant Q as RequestQueue
  participant C as DraftRequestConsumer
  participant W as RefinementWorkflow
  participant Wr as WriterAgent
  participant K as CriticAgent
  participant E as ArticleEntity
  participant V as ArticlesView

  U->>API: POST /api/articles {topic, qualityThreshold}
  API->>Q: append BriefSubmitted
  API-->>U: 202 {articleId}
  Q->>C: BriefSubmitted
  C->>W: start({articleId, topic, threshold, maxIterations=4})
  W->>E: emit ArticleCreated (DRAFTING)

  W->>Wr: DRAFT(topic, threshold)
  Wr-->>W: DraftPassage #1 (95 words)
  W->>E: emit AttemptDrafted (n=1)
  W->>E: status EVALUATING
  W->>K: EVALUATE(DraftPassage #1)
  K-->>W: Critique{REVISE, score=2, 3 bullets}
  W->>E: emit AttemptCritiqued (n=1, REVISE)

  W->>Wr: REVISE_DRAFT(topic, prior, notes)
  Wr-->>W: DraftPassage #2 (112 words)
  W->>E: emit AttemptDrafted (n=2)
  W->>K: EVALUATE(DraftPassage #2)
  K-->>W: Critique{ACCEPT, score=4, rationale}
  W->>E: emit AttemptCritiqued (n=2, ACCEPT)
  W->>E: emit ArticleAccepted (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ArticleEntity`

```mermaid
stateDiagram-v2
  [*] --> DRAFTING
  DRAFTING --> EVALUATING: AttemptDrafted
  EVALUATING --> DRAFTING: Critique = REVISE, iterations < max
  EVALUATING --> ACCEPTED: Critique = ACCEPT
  EVALUATING --> BUDGET_EXHAUSTED: Critique = REVISE, iterations = max
  ACCEPTED --> [*]
  BUDGET_EXHAUSTED --> [*]
```

## Entity model

```mermaid
erDiagram
  ArticleEntity ||--o{ ArticleCreated : emits
  ArticleEntity ||--o{ AttemptDrafted : emits
  ArticleEntity ||--o{ AttemptCritiqued : emits
  ArticleEntity ||--o{ ArticleAccepted : emits
  ArticleEntity ||--o{ BudgetExhausted : emits
  ArticleEntity ||--o{ EvalRecorded : emits
  ArticlesView }o--|| ArticleEntity : projects
  RequestQueue ||--o{ BriefSubmitted : emits
  DraftRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `WriterAgent` | `application/WriterAgent.java` |
| `CriticAgent` | `application/CriticAgent.java` |
| `WritingTasks` | `application/WritingTasks.java` |
| `RefinementWorkflow` | `application/RefinementWorkflow.java` |
| `ArticleEntity` | `application/ArticleEntity.java` (state in `domain/Article.java`, events in `domain/ArticleEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `ArticlesView` | `application/ArticlesView.java` |
| `DraftRequestConsumer` | `application/DraftRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `WritingEndpoint` | `api/WritingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `draftStep` and `critiqueStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(budgetExhaustedStep))` — the workflow degrades to `BUDGET_EXHAUSTED` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `WritingEndpoint.submit` uses `(topic, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(articleId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxIterations ceiling:** read from `self-critiquing-writer-loop.refinement.max-iterations` (default 4). The workflow checks the count BEFORE calling `draftStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best draft and every critique on the entity.
- **Budget-cap semantics:** when the iteration ceiling fires, `budgetExhaustedStep` emits `BudgetExhausted{iterationsUsed, bestAttemptNumber, bestText}` then the terminal `EvalRecorded` event. The halt is observable via the SSE stream and via `GET /api/articles/{id}`.
