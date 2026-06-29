# PLAN — multiformat-hybrid-rag

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
  classDef store fill:#0e1e1f,stroke:#22d3ee,color:#22d3ee;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Indexer[ChunkIndexer]:::cons
  Store[ChunkStore]:::store
  WF[QueryWorkflow]:::wf
  Agent[ResearchAgent]:::agent
  Guard[CitationGuardrail]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  Indexer -->|indexes| Store
  WF -->|retrieveStep topK| Store
  Store -->|RetrievedChunk list| WF
  WF -->|attachRetrieval| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|accept / reject| Agent
  Agent -->|ResearchAnswer| WF
  WF -->|recordAnswer| Entity
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
  participant WF as QueryWorkflow
  participant CS as ChunkStore
  participant A as ResearchAgent
  participant G as CitationGuardrail

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  API->>WF: start(queryId)
  WF->>CS: topK(question, 20)
  CS-->>WF: List<RetrievedChunk>
  WF->>E: attachRetrieval(result)
  WF->>E: markAnswering
  WF->>A: runSingleTask(question + chunk attachments)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>WF: ResearchAnswer
  WF->>E: recordAnswer(answer)
  E-.->>U: SSE event(ANSWERED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RETRIEVING: RetrievalCompleted
  RETRIEVING --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWERED: AnswerRecorded (ANSWERED)
  ANSWERING --> NO_RESULT: AnswerRecorded (NO_RESULT)
  ANSWERING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (retrieval error)
  ANSWERED --> [*]
  NO_RESULT --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ RetrievalCompleted : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  ChunkIndexer }o--|| ChunkStore : writes
  QueryWorkflow }o--|| ChunkStore : reads
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  ResearchAgent ||--o{ ResearchAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `ChunkIndexer` | `application/ChunkIndexer.java` |
| `ChunkStore` | `application/ChunkStore.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `ResearchAgent` | `application/ResearchAgent.java` (tasks in `application/QueryTasks.java`) |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `retrieveStep` 5 s (in-process keyword search, no IO), `answerStep` 60 s (LLM latency), `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates model response time (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; redelivered `QuerySubmitted` events reach `QueryEntity.submit` which is event-version-guarded — a duplicate submit on an already-submitted query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"agent-" + queryId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries.
- **Guardrail-driven retry**: when `CitationGuardrail` rejects a candidate answer, the rejection is returned as a structured error to the agent loop. Each rejection counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `answerStep` fails over to `error` and the entity transitions to `FAILED`.
- **Retrieval is synchronous and deterministic**: `ChunkStore.topK` runs in-process inside `retrieveStep`. No embedding call, no external service. The same question over the same corpus always returns the same ranked list.
- **No saga / no compensation**: every step is either a pure in-memory read, an append-only event write, or a single-task agent call. Nothing external to roll back.
