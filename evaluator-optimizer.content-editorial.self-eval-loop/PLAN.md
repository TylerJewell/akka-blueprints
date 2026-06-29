# PLAN — self-eval-loop

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

  Poet[PoetAgent]:::agent
  Critic[CriticAgent]:::agent

  WF[RefinementWorkflow]:::wf
  Post[PostEntity]:::ese
  Queue[RequestQueue]:::ese
  View[PostsView]:::view
  Consumer[DraftRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[PoetryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue brief| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|draft / revise| Poet
  WF -->|evaluate| Critic
  WF -->|emit events| Post
  Post -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Post
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PoetryEndpoint
  participant Q as RequestQueue
  participant C as DraftRequestConsumer
  participant W as RefinementWorkflow
  participant P as PoetAgent
  participant K as CriticAgent
  participant E as PostEntity
  participant V as PostsView

  U->>API: POST /api/posts {topic, ceiling}
  API->>Q: append BriefSubmitted
  API-->>U: 202 {postId}
  Q->>C: BriefSubmitted
  C->>W: start({postId, topic, ceiling, maxAttempts=4})
  W->>E: emit PostCreated (DRAFTING)

  W->>P: DRAFT(topic, ceiling)
  P-->>W: DraftPassage #1 (280 chars)
  W->>E: emit AttemptDrafted (n=1)
  Note over W: guardrailStep (deterministic length check)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>E: status EVALUATING
  W->>K: EVALUATE(DraftPassage #1)
  K-->>W: Critique{REVISE, score=3, 3 bullets}
  W->>E: emit AttemptCritiqued (n=1, REVISE)

  W->>P: REVISE_DRAFT(topic, prior, notes)
  P-->>W: DraftPassage #2 (260 chars)
  W->>E: emit AttemptDrafted (n=2)
  W->>E: emit AttemptGuardrailVerdictRecorded (passed=true)
  W->>K: EVALUATE(DraftPassage #2)
  K-->>W: Critique{ACCEPT, score=5, rationale}
  W->>E: emit AttemptCritiqued (n=2, ACCEPT)
  W->>E: emit PostAccepted (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `PostEntity`

```mermaid
stateDiagram-v2
  [*] --> DRAFTING
  DRAFTING --> EVALUATING: AttemptDrafted + guardrail passed
  DRAFTING --> DRAFTING: guardrail blocked, re-draft
  EVALUATING --> DRAFTING: Critique = REVISE, attempts < max
  EVALUATING --> ACCEPTED: Critique = ACCEPT
  EVALUATING --> REJECTED_FINAL: Critique = REVISE, attempts = max
  ACCEPTED --> [*]
  REJECTED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  PostEntity ||--o{ PostCreated : emits
  PostEntity ||--o{ AttemptDrafted : emits
  PostEntity ||--o{ AttemptGuardrailVerdictRecorded : emits
  PostEntity ||--o{ AttemptCritiqued : emits
  PostEntity ||--o{ PostAccepted : emits
  PostEntity ||--o{ PostRejectedFinal : emits
  PostEntity ||--o{ EvalRecorded : emits
  PostsView }o--|| PostEntity : projects
  RequestQueue ||--o{ BriefSubmitted : emits
  DraftRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PoetAgent` | `application/PoetAgent.java` |
| `CriticAgent` | `application/CriticAgent.java` |
| `PoetryTasks` | `application/PoetryTasks.java` |
| `RefinementWorkflow` | `application/RefinementWorkflow.java` |
| `PostEntity` | `application/PostEntity.java` (state in `domain/Post.java`, events in `domain/PostEvent.java`) |
| `RequestQueue` | `application/RequestQueue.java` |
| `PostsView` | `application/PostsView.java` |
| `DraftRequestConsumer` | `application/DraftRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `PoetryEndpoint` | `api/PoetryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `draftStep` and `critiqueStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(rejectStep))` — the workflow degrades to `REJECTED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `PoetryEndpoint.submit` uses `(topic, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(postId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `self-eval-loop.refinement.max-attempts` (default 4). The workflow checks the count BEFORE calling `draftStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the best draft and every critique on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it computes the character count from the draft and either advances to `critiqueStep` or returns to `draftStep` with a structured feedback note. The structured feedback never becomes an LLM-generated critique; it stays a deterministic `CritiqueNotes` payload with a single bullet.
