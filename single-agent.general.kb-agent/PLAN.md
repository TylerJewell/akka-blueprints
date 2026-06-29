# PLAN — kb-agent

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
  classDef index fill:#0e1a2a,stroke:#60A5FA,color:#60A5FA;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Retriever[PassageRetriever]:::cons
  DocConsumer[KbDocumentConsumer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[KbAnswerAgent]:::agent
  Scorer[GroundednessScorer]:::scorer
  Index[KbIndex]:::index
  View[KbView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|DocumentIndexed| DocConsumer
  DocConsumer -->|upsert| Index
  Entity -.->|QuerySubmitted| Retriever
  Retriever -->|retrieve| Index
  Retriever -->|attachPassages| Entity
  Retriever -->|start workflow| WF
  WF -->|awaitPassagesStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -->|KbAnswer| WF
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
  participant R as PassageRetriever
  participant W as QueryWorkflow
  participant A as KbAnswerAgent
  participant Sc as GroundednessScorer

  U->>API: POST /api/queries
  API->>E: submit(questionText, submittedBy)
  E-->>API: { queryId }
  E-.->>R: QuerySubmitted
  R->>R: TF-IDF retrieve top-5 passages
  R->>E: attachPassages(passages)
  R->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: passages.isPresent()
  W->>E: markAnswering
  W->>A: runSingleTask(question + passages attachment)
  A-->>W: KbAnswer
  W->>E: recordAnswer(answer)
  W->>Sc: score(answer, passages)
  Sc-->>W: GroundednessResult
  W->>E: recordGroundedness(groundedness)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RETRIEVING: PassagesAttached
  RETRIEVING --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWER_RECORDED: AnswerRecorded
  ANSWER_RECORDED --> EVALUATED: GroundednessScored
  ANSWERING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (retriever error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ PassagesAttached : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ GroundednessScored : emits
  QueryEntity ||--o{ QueryFailed : emits
  KbView }o--|| QueryEntity : projects
  PassageRetriever }o--|| QueryEntity : subscribes
  KbDocumentConsumer }o--|| KbIndex : updates
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  KbAnswerAgent ||--o{ KbAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `PassageRetriever` | `application/PassageRetriever.java` |
| `KbDocumentConsumer` | `application/KbDocumentConsumer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `KbAnswerAgent` | `application/KbAnswerAgent.java` (tasks in `application/KbTasks.java`) |
| `GroundednessScorer` | `application/GroundednessScorer.java` |
| `KbIndex` | `application/KbIndex.java` |
| `KbView` | `application/KbView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitPassagesStep` 15 s, `answerStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; `PassageRetriever` may redeliver `QuerySubmitted` events — `QueryEntity.attachPassages` is event-version-guarded and treats a second attach on an already-retrieving query as a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"answerer-" + queryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps internal retries.
- **KbIndex thread safety**: `KbIndex` is a singleton shared between `PassageRetriever` (reads) and `KbDocumentConsumer` (writes). Both operations synchronise on the index's internal lock; reads are non-blocking when no write is in progress.
- **Eval is synchronous and deterministic**: `GroundednessScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
