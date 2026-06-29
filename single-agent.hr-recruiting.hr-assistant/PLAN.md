# PLAN — hr-assistant

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
  Entity[QueryEntity]:::ese
  Sanitizer[QuerySanitizer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[HrPolicyAgent]:::agent
  Guard[PolicyAnswerGuardrail]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|QuerySubmitted| Sanitizer
  Sanitizer -->|PII pass + special-category pass| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|answerStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|PolicyAnswer| WF
  WF -->|recordAnswer| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Employee (UI)
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant S as QuerySanitizer
  participant W as QueryWorkflow
  participant A as HrPolicyAgent
  participant G as PolicyAnswerGuardrail

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  E-.->>S: QuerySubmitted
  S->>S: PII pass (redact emails / IDs / names)
  S->>S: special-category pass (redact health / religion / disability)
  S->>E: attachSanitized
  S->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: sanitized.isPresent()
  W->>E: markAnswering
  W->>A: runSingleTask(queryText + context attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: PolicyAnswer
  W->>E: recordAnswer(answer)
  E-.->>U: SSE event(ANSWER_RECORDED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: QuerySanitized
  SANITIZED --> ANSWERING: AnsweringStarted
  ANSWERING --> ANSWER_RECORDED: AnswerRecorded
  ANSWERING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (sanitizer error)
  ANSWER_RECORDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ QuerySanitized : emits
  QueryEntity ||--o{ AnsweringStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  QuerySanitizer }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  HrPolicyAgent ||--o{ PolicyAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/HrQuery.java`, events in `domain/QueryEvent.java`) |
| `QuerySanitizer` | `application/QuerySanitizer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `HrPolicyAgent` | `application/HrPolicyAgent.java` (tasks in `application/HrQueryTasks.java`) |
| `PolicyAnswerGuardrail` | `application/PolicyAnswerGuardrail.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `answerStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `answerStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; the `QuerySanitizer` Consumer is idempotent because `QueryEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"agent-" + queryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `PolicyAnswerGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. If all 3 iterations fail validation, the workflow's `answerStep` fails over to `error` and the entity transitions to `FAILED`.
- **Two-pass sanitizer in one Consumer**: both the PII pass and the special-category pass execute synchronously inside the same `QuerySanitizer.onEvent` handler, in order. There is no intermediate event between the two passes — `QuerySanitized` is emitted only once the combined `SanitizedQueryContext` is assembled.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
