# PLAN — notion-rag

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

  API[QueryEndpoint]:::ep
  Entity[SessionEntity]:::ese
  Retriever[NotionRetriever]:::cons
  WF[QueryWorkflow]:::wf
  Agent[NotionQueryAgent]:::agent
  Guard[GroundingGuardrail]:::guard
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|submitQuestion| Entity
  Entity -.->|QuestionSubmitted| Retriever
  Retriever -->|Notion API query| Retriever
  Retriever -->|attachRows| Entity
  Retriever -->|start workflow| WF
  WF -->|awaitRowsStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|QueryAnswer| WF
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
  participant E as SessionEntity
  participant R as NotionRetriever
  participant W as QueryWorkflow
  participant A as NotionQueryAgent
  participant G as GroundingGuardrail

  U->>API: POST /api/sessions/{id}/questions
  API->>E: submitQuestion(questionId, questionText)
  E-->>API: { questionId }
  E-.->>R: QuestionSubmitted
  R->>R: query Notion API
  R->>E: attachRows(questionId, retrievedRows)
  R->>W: start(questionId)
  W->>E: poll getSession
  E-->>W: question.status == ROWS_RETRIEVED
  W->>E: markAnswering
  W->>A: runSingleTask(question + rows attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept (all rowIds grounded)
  A-->>W: QueryAnswer
  W->>E: recordAnswer(questionId, answer)
  E-.->>U: SSE event(ANSWERED)
```

## State machine — `QuestionStatus` inside `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> ROWS_RETRIEVED: RowsRetrieved
  ROWS_RETRIEVED --> ANSWERING: AnswerStarted
  ANSWERING --> ANSWERED: AnswerRecorded
  ANSWERING --> FAILED: QuestionFailed (agent or guardrail exhaustion)
  SUBMITTED --> FAILED: QuestionFailed (retriever error)
  ANSWERED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionCreated : emits
  SessionEntity ||--o{ QuestionSubmitted : emits
  SessionEntity ||--o{ RowsRetrieved : emits
  SessionEntity ||--o{ AnswerStarted : emits
  SessionEntity ||--o{ AnswerRecorded : emits
  SessionEntity ||--o{ QuestionFailed : emits
  SessionView }o--|| SessionEntity : projects
  NotionRetriever }o--|| SessionEntity : subscribes
  QueryWorkflow }o--|| SessionEntity : reads-and-writes
  NotionQueryAgent ||--o{ QueryAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `NotionRetriever` | `application/NotionRetriever.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `NotionQueryAgent` | `application/NotionQueryAgent.java` (tasks in `application/QueryTasks.java`) |
| `GroundingGuardrail` | `application/GroundingGuardrail.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitRowsStep` 15 s, `answerStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + questionId` as the workflow id; `NotionRetriever` may redeliver `QuestionSubmitted` events but `SessionEntity.attachRows` is event-version-guarded — a second attach attempt against an already-retrieved question is a no-op.
- **One agent per question**: the AutonomousAgent instance id is `"agent-" + questionId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries.
- **Guardrail-driven retry**: when `GroundingGuardrail` rejects a candidate response, the agent loop counts one iteration toward `maxIterationsPerTask`. If all 3 fail, the workflow's `answerStep` fails over to `error` and the question transitions to `FAILED`.
- **Retriever is not an LLM**: `NotionRetriever` is a Consumer that calls the Notion HTTP API. The single-agent invariant is preserved — only `NotionQueryAgent` calls a model.
- **No saga / no compensation**: every step is a pure read, an append-only event write, or a single-task agent call. Nothing external needs rolling back.
