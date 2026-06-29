# PLAN — code-search-rag-agent

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
  classDef scorer fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Sanitizer[ChunkSanitizer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[CodeSearchAgent]:::agent
  Scorer[GroundingScorer]:::scorer
  View[ChunkIndexView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|QuerySubmitted| Sanitizer
  Sanitizer -->|attachSanitizedChunks| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -->|SearchAnswer| WF
  WF -->|recordAnswer| Entity
  WF -->|groundingStep score| Scorer
  Scorer -->|GroundingResult| WF
  WF -->|recordGrounding| Entity
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
  participant S as ChunkSanitizer
  participant W as QueryWorkflow
  participant A as CodeSearchAgent
  participant Sc as GroundingScorer

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  E-.->>S: QuerySubmitted
  S->>S: redact secrets from chunks
  S->>E: attachSanitizedChunks
  S->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: sanitized.isPresent()
  W->>E: markAnswering
  W->>A: runSingleTask(question + chunk attachments)
  A-->>W: SearchAnswer
  W->>E: recordAnswer(answer)
  W->>Sc: score(answer, sanitized.chunks)
  Sc-->>W: GroundingResult
  W->>E: recordGrounding(grounding)
  E-.->>U: SSE event(GROUNDED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> CHUNKS_SANITIZED: ChunksSanitized
  CHUNKS_SANITIZED --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWERED: AnswerRecorded
  ANSWERED --> GROUNDED: GroundingScored
  ANSWERING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (sanitizer error)
  GROUNDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ ChunksSanitized : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ GroundingScored : emits
  QueryEntity ||--o{ QueryFailed : emits
  ChunkIndexView }o--|| QueryEntity : projects
  ChunkSanitizer }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  CodeSearchAgent ||--o{ SearchAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `ChunkSanitizer` | `application/ChunkSanitizer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `CodeSearchAgent` | `application/CodeSearchAgent.java` (tasks in `application/QueryTasks.java`) |
| `GroundingScorer` | `application/GroundingScorer.java` |
| `ChunkIndexView` | `application/ChunkIndexView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `answerStep` 60 s, `groundingStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency with multiple chunk attachments (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; the `ChunkSanitizer` Consumer is allowed to redeliver `QuerySubmitted` events because `QueryEntity.attachSanitizedChunks` is event-version-guarded — a second sanitize attempt against an already-sanitized query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"searcher-" + queryId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` limits retries.
- **Grounding is synchronous and deterministic**: `GroundingScorer` runs in-process inside `groundingStep`. No LLM call, no external service — the same answer + chunks always produce the same score. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
- **Chunk attachment fan-out**: `answerStep` attaches one file per chunk (`chunkId.txt`). If the sanitized chunk list is empty (no chunks matched the corpus tag), the agent receives the question with zero attachments and is expected to return a "not found" answer.
