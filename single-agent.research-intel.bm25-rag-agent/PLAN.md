# PLAN — bm25-rag-agent

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
  classDef idx fill:#0e1e1e,stroke:#22d3ee,color:#22d3ee;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Retriever[BM25Retriever]:::cons
  Index[CorpusIndex]:::idx
  WF[QueryWorkflow]:::wf
  Agent[CorpusQueryAgent]:::agent
  Guard[CitationGuardrail]:::guard
  Scorer[GroundingScorer]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|QuerySubmitted| Retriever
  Retriever -->|BM25 search| Index
  Index -->|PassageRef list| Retriever
  Retriever -->|attachPassages| Entity
  Retriever -->|start workflow| WF
  WF -->|awaitPassagesStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|QueryAnswer| WF
  WF -->|recordAnswer| Entity
  WF -->|scoringStep score| Scorer
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
  participant R as BM25Retriever
  participant I as CorpusIndex
  participant W as QueryWorkflow
  participant A as CorpusQueryAgent
  participant G as CitationGuardrail
  participant Sc as GroundingScorer

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  E-.->>R: QuerySubmitted
  R->>I: search(questionText, topK)
  I-->>R: List<PassageRef>
  R->>E: attachPassages(passages)
  R->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: passages.isPresent()
  W->>E: markAnswering
  W->>A: runSingleTask(question + attachments)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: QueryAnswer
  W->>E: recordAnswer(answer)
  W->>Sc: score(answer, passages)
  Sc-->>W: GroundingResult
  W->>E: recordGrounding(grounding)
  E-.->>U: SSE event(SCORED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PASSAGES_ATTACHED: PassagesAttached
  PASSAGES_ATTACHED --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWER_RECORDED: AnswerRecorded
  ANSWER_RECORDED --> SCORED: GroundingScored
  ANSWERING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (retriever error)
  SCORED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ PassagesAttached : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ GroundingScored : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  BM25Retriever }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  CorpusQueryAgent ||--o{ QueryAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `BM25Retriever` | `application/BM25Retriever.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `CorpusQueryAgent` | `application/CorpusQueryAgent.java` (tasks in `application/QueryTasks.java`) |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `GroundingScorer` | `application/GroundingScorer.java` |
| `CorpusIndex` | `application/CorpusIndex.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitPassagesStep` 15 s, `answerStep` 60 s, `scoringStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; the `BM25Retriever` Consumer is allowed to redeliver `QuerySubmitted` events because `QueryEntity.attachPassages` is event-version-guarded — a second attach attempt against an already-passaged query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"agent-" + queryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `CitationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `answerStep` fails over to `error` and the entity transitions to `FAILED`.
- **Scoring is synchronous and deterministic**: `GroundingScorer` runs in-process inside `scoringStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent guarantee.
- **Index is built once**: `CorpusIndex` is loaded and indexed from `passages.jsonl` at application startup. It is not rebuilt per query; all `BM25Retriever` invocations share the same read-only index.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
