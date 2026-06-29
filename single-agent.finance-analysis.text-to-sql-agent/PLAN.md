# PLAN — text-to-sql-agent

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
  Sanitizer[ResultSanitizer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[SqlGeneratorAgent]:::agent
  Guard[SqlGuardrail]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep
  DB[(receipts.db)]:::ese

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|generateStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|pass / reject| Agent
  Agent -->|execute_sql| DB
  DB -->|rows| Agent
  Agent -->|QueryResult| WF
  WF -->|recordResult| Entity
  Entity -.->|QueryExecuted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  WF -->|awaitSanitizedStep poll| Entity
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
  participant A as SqlGeneratorAgent
  participant G as SqlGuardrail
  participant DB as receipts.db
  participant S as ResultSanitizer

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  API->>W: start(queryId)
  W->>A: runSingleTask(question + schema attachment)
  A->>G: before-tool-call(execute_sql, sql)
  G-->>A: pass
  A->>DB: execute_sql(SELECT ...)
  DB-->>A: result rows
  A-->>W: QueryResult (generatedSql + rows + summary)
  W->>E: recordResult(rawResult)
  E-.->>S: QueryExecuted
  S->>S: redact PII from rows
  S->>E: attachSanitized
  W->>E: poll getQuery
  E-->>W: sanitized.isPresent()
  E-.->>U: SSE event(SANITIZED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> GENERATING: SqlGenerated
  GENERATING --> EXECUTING: ExecutionStarted
  EXECUTING --> EXECUTED: QueryExecuted
  EXECUTED --> SANITIZED: ResultSanitized
  GENERATING --> FAILED: QueryFailed (guardrail-exhausted)
  EXECUTING --> FAILED: QueryFailed (db error)
  SUBMITTED --> FAILED: QueryFailed (agent error)
  SANITIZED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuestionSubmitted : emits
  QueryEntity ||--o{ SqlGenerated : emits
  QueryEntity ||--o{ ExecutionStarted : emits
  QueryEntity ||--o{ QueryExecuted : emits
  QueryEntity ||--o{ ResultSanitized : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  ResultSanitizer }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  SqlGeneratorAgent ||--o{ QueryResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/QueryResult.java`, events in `domain/QueryEvent.java`) |
| `ResultSanitizer` | `application/ResultSanitizer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `SqlGeneratorAgent` | `application/SqlGeneratorAgent.java` (tasks in `application/QueryTasks.java`) |
| `SqlGuardrail` | `application/SqlGuardrail.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `generateStep` 90 s, `awaitSanitizedStep` 15 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 90 s on `generateStep` accommodates LLM latency plus tool-call round-trips (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; the `ResultSanitizer` Consumer is allowed to redeliver `QueryExecuted` events because `QueryEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"sql-" + queryId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4 — one extra over the legal-compliance blueprint because a SQL guardrail rejection is highly likely to require two internal tries before the agent produces a purely safe SELECT.
- **Guardrail-driven retry**: when `SqlGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations are rejected, the workflow's `generateStep` fails over to `error` and the entity transitions to `FAILED`.
- **Sanitizer is asynchronous**: `ResultSanitizer` runs as a separate Consumer after `QueryExecuted`. `awaitSanitizedStep` polls `QueryEntity` every 1 s up to 15 s, advancing when `query.sanitized().isPresent()` returns true. This keeps the agent loop's wall-clock bounded and makes the sanitizer independently replaceable.
- **SQLite is embedded**: `receipts.db` is opened with `DriverManager.getConnection("jdbc:sqlite::resource:receipts.db")` using the sqlite-jdbc driver on the classpath. No separate process, no network call.
- **No saga / no compensation**: query execution is read-only; there is nothing to roll back.
