# PLAN — rag-citation-answerer

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
  classDef store fill:#0e1e1a,stroke:#22d3ee,color:#22d3ee;

  API[QueryEndpoint]:::ep
  DocEntity[DocumentEntity]:::ese
  QueryEntity[QueryEntity]:::ese
  Sanitizer[DocumentSanitizer]:::cons
  Indexer[ChunkIndexer]:::cons
  Store[ChunkStore]:::store
  WF[AnswerWorkflow]:::wf
  Agent[CitationAnswererAgent]:::agent
  Guard[CitationGuardrail]:::guard
  DocView[DocumentView]:::view
  QueryView[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|upload| DocEntity
  API -->|ask| QueryEntity
  API -->|start workflow| WF
  DocEntity -.->|DocumentUploaded| Sanitizer
  Sanitizer -->|attachSanitized| DocEntity
  DocEntity -.->|DocumentSanitized| Indexer
  Indexer -->|store chunks| Store
  Indexer -->|markIndexed| DocEntity
  WF -->|retrieveStep| Store
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|CitedAnswer| WF
  WF -->|recordAnswer| QueryEntity
  DocEntity -.->|projects| DocView
  QueryEntity -.->|projects| QueryView
  API -->|list/SSE| DocView
  API -->|list/SSE| QueryView
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as QueryEndpoint
  participant DE as DocumentEntity
  participant S as DocumentSanitizer
  participant CI as ChunkIndexer
  participant CS as ChunkStore
  participant QE as QueryEntity
  participant W as AnswerWorkflow
  participant A as CitationAnswererAgent
  participant G as CitationGuardrail

  U->>API: POST /api/documents
  API->>DE: upload(request)
  DE-->>API: { documentId }
  DE-.->>S: DocumentUploaded
  S->>S: redact PII
  S->>DE: attachSanitized
  DE-.->>CI: DocumentSanitized
  CI->>CI: split into chunks
  CI->>CS: store chunks
  CI->>DE: markIndexed(chunkCount)
  U->>API: POST /api/queries
  API->>QE: submit(request)
  API->>W: start(queryId)
  QE-->>API: { queryId }
  W->>CS: retrieve(documentIds, questionText, topK=5)
  CS-->>W: RetrievalResult
  W->>QE: markRetrieving
  W->>A: runSingleTask(question + chunks attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: CitedAnswer
  W->>QE: recordAnswer(answer)
  QE-.->>U: SSE event(ANSWERED)
```

## State machine — `DocumentEntity`

```mermaid
stateDiagram-v2
  [*] --> UPLOADED
  UPLOADED --> SANITIZED: DocumentSanitized
  SANITIZED --> CHUNKED: DocumentChunked
  CHUNKED --> INDEXED: DocumentIndexed
  UPLOADED --> FAILED: DocumentFailed (sanitizer error)
  SANITIZED --> FAILED: DocumentFailed (indexer error)
  INDEXED --> [*]
  FAILED --> [*]
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RETRIEVING: RetrievalStarted
  RETRIEVING --> ANSWERED: AnswerRecorded
  RETRIEVING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (retrieval error)
  ANSWERED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  DocumentEntity ||--o{ DocumentUploaded : emits
  DocumentEntity ||--o{ DocumentSanitized : emits
  DocumentEntity ||--o{ DocumentChunked : emits
  DocumentEntity ||--o{ DocumentIndexed : emits
  DocumentEntity ||--o{ DocumentFailed : emits
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ RetrievalStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  DocumentView }o--|| DocumentEntity : projects
  QueryView }o--|| QueryEntity : projects
  DocumentSanitizer }o--|| DocumentEntity : subscribes
  ChunkIndexer }o--|| DocumentEntity : subscribes
  ChunkStore }o--|| ChunkIndexer : populated-by
  AnswerWorkflow }o--|| ChunkStore : reads
  AnswerWorkflow }o--|| QueryEntity : reads-and-writes
  CitationAnswererAgent ||--o{ CitedAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DocumentEntity` | `application/DocumentEntity.java` (state in `domain/Document.java`, events in `domain/DocumentEvent.java`) |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `DocumentSanitizer` | `application/DocumentSanitizer.java` |
| `ChunkIndexer` | `application/ChunkIndexer.java` |
| `ChunkStore` | `application/ChunkStore.java` |
| `AnswerWorkflow` | `application/AnswerWorkflow.java` |
| `CitationAnswererAgent` | `application/CitationAnswererAgent.java` (tasks in `application/QueryTasks.java`) |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `DocumentView` | `application/DocumentView.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `retrieveStep` 10 s, `answerStep` 90 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AnswerWorkflow::error)`. The 90 s on `answerStep` accommodates LLM latency at the upper tail (Lesson 4).
- **Document readiness**: `AnswerWorkflow` starts only after the query is submitted; it does not wait for all selected documents to be indexed. If a selected `documentId` has no indexed chunks, `ChunkStore.retrieve` returns an empty list for that document — the workflow proceeds with whatever chunks are available, and the answer's `confidence` reflects the sparse retrieval.
- **Idempotency**: `DocumentSanitizer` is allowed to redeliver `DocumentUploaded` events; `DocumentEntity.attachSanitized` is event-version-guarded. `ChunkIndexer` re-indexing the same document overwrites existing chunk entries in `ChunkStore` (same chunkIds, same content — idempotent).
- **One agent per query**: the AutonomousAgent instance id is `"answerer-" + queryId`. Each task has its own conversation context. `maxIterationsPerTask(3)` caps guardrail-triggered retries.
- **Guardrail-driven retry**: when `CitationGuardrail` rejects a candidate, the agent loop counts one iteration toward `maxIterationsPerTask`. All 3 failing means the workflow's `answerStep` fails over to `error` and the entity transitions to `FAILED`.
- **No LLM retrieval**: `ChunkStore` uses keyword-TF scoring — no embedding model call, no external vector database. This is intentional for the out-of-the-box tier; a deployer wires a real embedding-based retriever by replacing `ChunkStore.retrieve` internals.
- **Raw text isolation**: `DocumentEntity` retains `upload.rawText` for audit. `ChunkStore` stores only chunks derived from `sanitized.redactedText`. The retrieval path can never return raw PII text.
