# PLAN — chroma-rag-agent

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
  classDef infra fill:#111827,stroke:#6b7280,color:#9ca3af;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Indexer[CorpusIndexer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[RagAgent]:::agent
  Guard[CitationGuardrail]:::guard
  Scorer[GroundednessScorer]:::guard
  View[RagView]:::view
  App[AppEndpoint]:::ep
  Chroma[(ChromaDB embedded)]:::infra

  API -->|submit| Entity
  API -->|startWorkflow| WF
  Entity -.->|CorpusLoadRequested| Indexer
  Indexer -->|embed + upsert| Chroma
  Indexer -->|corpusIndexed| Entity
  WF -->|retrieveStep query| Chroma
  Chroma -->|top-k chunks| WF
  WF -->|recordChunks| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|RagAnswer| WF
  WF -->|recordAnswer| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|GroundednessResult| WF
  WF -->|recordGroundedness| Entity
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
  participant W as QueryWorkflow
  participant Ch as ChromaDB
  participant A as RagAgent
  participant G as CitationGuardrail
  participant Sc as GroundednessScorer

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  API->>W: start(queryId)
  W->>Ch: query("docs", questionText, k=5)
  Ch-->>W: List<RetrievedChunk>
  W->>E: recordChunks(chunks)
  W->>E: markAnswering
  W->>A: runSingleTask(question + chunks JSON)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: RagAnswer
  W->>E: recordAnswer(answer)
  W->>Sc: score(answer, retrievedChunks)
  Sc-->>W: GroundednessResult
  W->>E: recordGroundedness(groundedness)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RETRIEVING: ChunksRetrieved
  RETRIEVING --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWERED: AnswerRecorded
  ANSWERED --> EVALUATED: GroundednessScored
  RETRIEVING --> FAILED: QueryFailed (retrieval error)
  ANSWERING --> FAILED: QueryFailed (agent error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ ChunksRetrieved : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ GroundednessScored : emits
  QueryEntity ||--o{ QueryFailed : emits
  RagView }o--|| QueryEntity : projects
  CorpusIndexer }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  RagAgent ||--o{ RagAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `CorpusIndexer` | `application/CorpusIndexer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `RagAgent` | `application/RagAgent.java` (tasks in `application/RagTasks.java`) |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `GroundednessScorer` | `application/GroundednessScorer.java` |
| `ChromaDbClient` | `application/ChromaDbClient.java` |
| `RagView` | `application/RagView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `retrieveStep` 20 s, `answerStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; the `QueryEndpoint` starts the workflow immediately after `submit`; duplicate submissions from the UI are prevented by the entity's command guard on `SUBMITTED` status.
- **One agent per query**: the AutonomousAgent instance id is `"rag-" + queryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `CitationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `answerStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `GroundednessScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent guarantee.
- **ChromaDB is embedded**: `ChromaDbClient` wraps the in-process Java client; no network hop, no external daemon. The corpus is indexed once at startup via `CorpusIndexer`.
- **No saga / no compensation**: every step is either a ChromaDB read, append-only event write, or a single-task agent call. There is nothing external to roll back.
