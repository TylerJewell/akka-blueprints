# PLAN — rag-baseline

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
  classDef eval fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Retriever[PassageRetriever]:::cons
  VS[VectorStore]:::eval
  WF[QueryWorkflow]:::wf
  Agent[AnswerAgent]:::agent
  Evaluator[GroundednessEvaluator]:::eval
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|QuerySubmitted| Retriever
  Retriever -->|search| VS
  VS -->|passages| Retriever
  Retriever -->|attachPassages| Entity
  Retriever -->|start workflow| WF
  WF -->|awaitPassagesStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -->|GroundedAnswer| WF
  WF -->|recordAnswer| Entity
  WF -->|evalStep score| Evaluator
  Evaluator -->|EvalResult| WF
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
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant R as PassageRetriever
  participant VS as VectorStore
  participant W as QueryWorkflow
  participant A as AnswerAgent
  participant Ev as GroundednessEvaluator

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  E-.->>R: QuerySubmitted
  R->>VS: search(questionText, topK=5)
  VS-->>R: List<Passage>
  R->>E: attachPassages
  R->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: retrieved.isPresent()
  W->>E: markAnswering
  W->>A: runSingleTask(question + passage attachments)
  A-->>W: GroundedAnswer
  W->>E: recordAnswer(answer)
  W->>Ev: score(answer, retrieved)
  Ev-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PASSAGES_RETRIEVED: PassagesRetrieved
  PASSAGES_RETRIEVED --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWER_RECORDED: AnswerRecorded
  ANSWER_RECORDED --> EVALUATED: EvaluationScored
  ANSWERING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (retriever error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ PassagesRetrieved : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ EvaluationScored : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  PassageRetriever }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  AnswerAgent ||--o{ GroundedAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `PassageRetriever` | `application/PassageRetriever.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `AnswerAgent` | `application/AnswerAgent.java` (tasks in `application/QueryTasks.java`) |
| `GroundednessEvaluator` | `application/GroundednessEvaluator.java` |
| `VectorStore` | `application/VectorStore.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitPassagesStep` 15 s, `answerStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; the `PassageRetriever` Consumer is allowed to redeliver `QuerySubmitted` events because `QueryEntity.attachPassages` is event-version-guarded — a second retrieval attempt against an already-retrieved query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"answerer-" + queryId`, giving each task its own conversation context. `maxIterationsPerTask(2)` caps any retry within the agent loop.
- **Eval is synchronous and deterministic**: `GroundednessEvaluator` runs in-process inside `evalStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent guarantee.
- **VectorStore is read-only at runtime**: loaded once from the classpath on startup; no writes during query processing. Thread-safe for concurrent reads.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. Nothing external to roll back.
