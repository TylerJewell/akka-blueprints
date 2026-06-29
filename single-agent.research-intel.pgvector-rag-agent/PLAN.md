# PLAN — pgvector-rag-agent

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
  classDef store fill:#0e1a2a,stroke:#38bdf8,color:#38bdf8;

  API[QueryEndpoint]:::ep
  CorpusE[CorpusEntity]:::ese
  QuestionE[QuestionEntity]:::ese
  Ingester[CorpusIngestionConsumer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[CorpusQueryAgent]:::agent
  Guard[CitationGuardrail]:::guard
  Scorer[AnswerEvaluationScorer]:::guard
  View[QuestionView]:::view
  App[AppEndpoint]:::ep
  PGV[(pgvector)]:::store

  API -->|add document| CorpusE
  API -->|submit question| QuestionE
  API -->|start workflow| WF
  CorpusE -.->|DocumentAdded| Ingester
  Ingester -->|embed + store chunks| PGV
  Ingester -->|markIndexed| CorpusE
  WF -->|embedStep embed question| PGV
  WF -->|retrieveStep cosine query| PGV
  WF -->|markRetrieving / markAnswering| QuestionE
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|Answer| WF
  WF -->|recordAnswer| QuestionE
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| QuestionE
  QuestionE -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as QueryEndpoint
  participant QE as QuestionEntity
  participant W as QueryWorkflow
  participant PGV as pgvector
  participant A as CorpusQueryAgent
  participant G as CitationGuardrail
  participant Sc as AnswerEvaluationScorer

  U->>API: POST /api/questions
  API->>QE: submit(questionText, askedBy)
  QE-->>API: { questionId }
  API->>W: start(questionId)
  W->>PGV: embed(questionText)
  PGV-->>W: float[] embedding
  W->>QE: markRetrieving
  W->>PGV: queryNearest(embedding, k=5)
  PGV-->>W: List<RetrievedPassage>
  W->>QE: markAnswering
  W->>A: runSingleTask(question + passages attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: Answer
  W->>QE: recordAnswer(answer)
  W->>Sc: score(answer, passages)
  Sc-->>W: EvalResult
  W->>QE: recordEvaluation(eval)
  QE-.->>U: SSE event(EVALUATED)
```

## State machine — `QuestionEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RETRIEVING: RetrievalStarted
  RETRIEVING --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWERED: AnswerRecorded
  ANSWERED --> EVALUATED: EvaluationScored
  ANSWERING --> FAILED: QuestionFailed (agent error)
  RETRIEVING --> FAILED: QuestionFailed (retrieval error)
  SUBMITTED --> FAILED: QuestionFailed (embed error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QuestionEntity ||--o{ QuestionSubmitted : emits
  QuestionEntity ||--o{ RetrievalStarted : emits
  QuestionEntity ||--o{ AnsweringStarted : emits
  QuestionEntity ||--o{ AnswerRecorded : emits
  QuestionEntity ||--o{ EvaluationScored : emits
  QuestionEntity ||--o{ QuestionFailed : emits
  CorpusEntity ||--o{ DocumentAdded : emits
  CorpusEntity ||--o{ DocumentIndexed : emits
  CorpusEntity ||--o{ IndexingFailed : emits
  QuestionView }o--|| QuestionEntity : projects
  CorpusIngestionConsumer }o--|| CorpusEntity : subscribes
  QueryWorkflow }o--|| QuestionEntity : reads-and-writes
  CorpusQueryAgent ||--o{ Answer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QuestionEntity` | `application/QuestionEntity.java` (state in `domain/Question.java`, events in `domain/QuestionEvent.java`) |
| `CorpusEntity` | `application/CorpusEntity.java` (state in `domain/CorpusDocument.java`, events in `domain/CorpusEvent.java`) |
| `CorpusIngestionConsumer` | `application/CorpusIngestionConsumer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `CorpusQueryAgent` | `application/CorpusQueryAgent.java` (tasks in `application/QueryTasks.java`) |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `AnswerEvaluationScorer` | `application/AnswerEvaluationScorer.java` |
| `EmbeddingService` | `application/EmbeddingService.java` |
| `PgVectorStore` | `application/PgVectorStore.java` |
| `ChunkSplitter` | `application/ChunkSplitter.java` |
| `QuestionView` | `application/QuestionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `embedStep` 10 s, `retrieveStep` 15 s, `answerStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + questionId` as the workflow id. `CorpusIngestionConsumer` may redeliver `DocumentAdded` events; `CorpusEntity.markIndexed` is event-version-guarded — a second indexing attempt against an already-indexed document is a no-op.
- **One agent per question**: the AutonomousAgent instance id is `"querier-" + questionId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `CitationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `answerStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `AnswerEvaluationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent guarantee.
- **pgvector connectivity**: the `PgVectorStore` acquires a connection from a `HikariCP` pool configured in `application.conf`. If pgvector is not reachable during `retrieveStep`, the step returns a failure and the workflow transitions the entity to `FAILED` — it does not hang indefinitely.
- **No saga / no compensation**: every step is either a pure compute, a pgvector read/write, or a single-task agent call. There is nothing external to roll back beyond what pgvector manages transactionally.
