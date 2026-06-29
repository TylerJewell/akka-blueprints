# PLAN — corrective-rag-workflow

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

  Retriever[RetrieverAgent]:::agent
  Evaluator[RelevanceEvaluatorAgent]:::agent
  WebSearch[WebSearchAgent]:::agent
  Synthesizer[AnswerSynthesizerAgent]:::agent

  WF[CorrectiveRagWorkflow]:::wf
  Query[QueryEntity]:::ese
  Corpus[CorpusStore]:::ese
  View[QueryView]:::view
  Consumer[QueryConsumer]:::cons
  Sim[QuerySimulator]:::ta
  Eval[EvalSampler]:::ta
  API[RagEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit query| Query
  Sim -.->|every 60s| API
  Query -.->|QueryCreated| Consumer
  Consumer -->|start workflow| WF
  WF -->|fetch chunks| Retriever
  Retriever -->|fetchChunks| Corpus
  WF -->|score relevance| Evaluator
  WF -->|web fallback| WebSearch
  WF -->|synthesize| Synthesizer
  WF -->|emit events| Query
  Query -.->|projects| View
  API -->|query / SSE| View
  Eval -.->|every 30s| Query
```

## Interaction sequence — J2 (corrective fallback path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as RagEndpoint
  participant Q as QueryEntity
  participant C as QueryConsumer
  participant W as CorrectiveRagWorkflow
  participant R as RetrieverAgent
  participant CS as CorpusStore
  participant EV as RelevanceEvaluatorAgent
  participant WS as WebSearchAgent
  participant SY as AnswerSynthesizerAgent
  participant V as QueryView

  U->>API: POST /api/queries {question}
  API->>Q: emit QueryCreated (RETRIEVING)
  API-->>U: 202 {queryId}
  Q->>C: QueryCreated
  C->>W: start({queryId, question, threshold=0.7, maxAttempts=3})

  W->>R: RETRIEVE(question)
  R->>CS: fetchChunks(question)
  CS-->>R: top-5 chunks (low match)
  R-->>W: RetrievalResult (2 chunks, scores 0.3–0.4)
  W->>Q: emit RetrievalAttempted (n=1)
  W->>Q: status EVALUATING

  W->>EV: EVALUATE_RELEVANCE(chunks, question)
  EV-->>W: RelevanceVerdict{INSUFFICIENT, avg=0.35}
  W->>Q: emit RelevanceVerdictRecorded (n=1, INSUFFICIENT)

  Note over W: guardrailStep (pure-function pattern check)
  W->>Q: emit GuardrailVerdictRecorded (cleared=true)
  W->>Q: status WEB_SEARCHING

  W->>WS: WEB_SEARCH(question)
  WS-->>W: WebSearchResult (4 results)
  W->>Q: emit WebSearchCompleted (n=1)

  W->>SY: SYNTHESIZE(chunks + webResults, question)
  SY-->>W: Answer{text, citedUrls, RETRIEVAL_PLUS_WEB}
  W->>Q: emit AnswerGenerated
  W->>Q: emit QueryAnswered (ANSWERED)
  Q-->>V: project
  V-->>U: SSE update
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> RETRIEVING
  RETRIEVING --> EVALUATING: RetrievalAttempted
  EVALUATING --> RETRIEVING: INSUFFICIENT, attempts < max
  EVALUATING --> WEB_SEARCHING: INSUFFICIENT, guardrail cleared
  EVALUATING --> ANSWERED: SUFFICIENT → synthesize
  WEB_SEARCHING --> ANSWERED: WebSearch + synthesize complete
  EVALUATING --> ANSWER_DEGRADED: INSUFFICIENT, guardrail blocked OR attempts exhausted
  WEB_SEARCHING --> ANSWER_DEGRADED: agent failure (defaultStepRecovery)
  ANSWERED --> [*]
  ANSWER_DEGRADED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ RetrievalAttempted : emits
  QueryEntity ||--o{ RelevanceVerdictRecorded : emits
  QueryEntity ||--o{ GuardrailVerdictRecorded : emits
  QueryEntity ||--o{ WebSearchCompleted : emits
  QueryEntity ||--o{ AnswerGenerated : emits
  QueryEntity ||--o{ QueryAnswered : emits
  QueryEntity ||--o{ QueryDegraded : emits
  QueryEntity ||--o{ EvalRecorded : emits
  QueryView }o--|| QueryEntity : projects
  CorpusStore ||--o{ DocumentChunk : holds
  RetrieverAgent }o--|| CorpusStore : fetchChunks
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RetrieverAgent` | `application/RetrieverAgent.java` |
| `RelevanceEvaluatorAgent` | `application/RelevanceEvaluatorAgent.java` |
| `WebSearchAgent` | `application/WebSearchAgent.java` |
| `AnswerSynthesizerAgent` | `application/AnswerSynthesizerAgent.java` |
| `RagTasks` | `application/RagTasks.java` |
| `CorrectiveRagWorkflow` | `application/CorrectiveRagWorkflow.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `CorpusStore` | `application/CorpusStore.java` |
| `QueryView` | `application/QueryView.java` |
| `QueryConsumer` | `application/QueryConsumer.java` |
| `QuerySimulator` | `application/QuerySimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `RagEndpoint` | `api/RagEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `retrieveStep`, `evaluateStep`, `webSearchStep`, and `synthesizeStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The guardrail step is in-process and effectively instant.
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(degradeStep))` — any unrecoverable agent failure ends in `ANSWER_DEGRADED`, not a hung workflow.
- **Idempotency:** `RagEndpoint.submit` uses `(question, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(queryId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxRetrievalAttempts ceiling:** read from `corrective-rag.retrieval.max-attempts` (default 3). The workflow checks the count BEFORE looping back to `retrieveStep`; it never recurses past the ceiling.
- **Guardrail placement:** `guardrailStep` runs only on the `INSUFFICIENT` branch, immediately before `webSearchStep`. A `SUFFICIENT` verdict never touches the guardrail.
- **CorpusStore seeding:** `CorpusStore` is pre-seeded on first start from `corpus-chunks.jsonl` via Bootstrap. Subsequent starts are idempotent because the entity ignores `AddChunk` commands for chunkIds it already holds.
