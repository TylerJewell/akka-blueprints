# PLAN — nl2sql

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
  classDef db fill:#0d1a2a,stroke:#60a5fa,color:#60a5fa;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  SchemaConsumer[SchemaRegistryConsumer]:::cons
  WF[QueryWorkflow]:::wf
  Agent[SqlQueryAgent]:::agent
  Guard[SqlSafetyGuardrail]:::guard
  Halt[SqlSafetyHalt]:::guard
  Scorer[QueryResultScorer]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep
  PG[(PostgreSQL)]:::db

  API -->|submit| Entity
  Entity -.->|QuerySubmitted| SchemaConsumer
  SchemaConsumer -->|attachSchema| Entity
  SchemaConsumer -->|start workflow| WF
  WF -->|awaitSchemaStep poll| Entity
  WF -->|translateAndRunStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -.->|safety-halt check| Halt
  Agent -->|QueryExecutionTool| PG
  PG -->|rows| Agent
  Agent -->|QueryResult| WF
  WF -->|recordResult| Entity
  WF -->|scoreStep| Scorer
  Scorer -->|ScoreResult| WF
  WF -->|recordScore| Entity
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
  participant SC as SchemaRegistryConsumer
  participant W as QueryWorkflow
  participant A as SqlQueryAgent
  participant G as SqlSafetyGuardrail
  participant PG as PostgreSQL
  participant Sc as QueryResultScorer

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  E-.->>SC: QuerySubmitted
  SC->>SC: lookup schema fragment
  SC->>E: attachSchema
  SC->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: schema.isPresent()
  W->>E: markTranslating
  W->>A: runSingleTask(question + schema)
  A->>G: before-tool-call(sql)
  G-->>A: accept
  A->>PG: QueryExecutionTool(SELECT ...)
  PG-->>A: rows
  A-->>W: QueryResult
  W->>E: recordResult(result)
  W->>Sc: score(result, schema)
  Sc-->>W: ScoreResult
  W->>E: recordScore(score)
  E-.->>U: SSE event(SCORED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SCHEMA_ATTACHED: SchemaAttached
  SCHEMA_ATTACHED --> TRANSLATING: TranslationStarted
  TRANSLATING --> RESULT_READY: ResultRecorded
  RESULT_READY --> SCORED: QueryScored
  TRANSLATING --> HALTED: QueryHalted (write/DDL detected)
  TRANSLATING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (schema error)
  SCORED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ SchemaAttached : emits
  QueryEntity ||--o{ TranslationStarted : emits
  QueryEntity ||--o{ ResultRecorded : emits
  QueryEntity ||--o{ QueryScored : emits
  QueryEntity ||--o{ QueryHalted : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  SchemaRegistryConsumer }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  SqlQueryAgent ||--o{ QueryResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `SchemaRegistryConsumer` | `application/SchemaRegistryConsumer.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `SqlQueryAgent` | `application/SqlQueryAgent.java` (tasks in `application/QueryTasks.java`) |
| `SqlSafetyGuardrail` | `application/SqlSafetyGuardrail.java` |
| `SqlSafetyHalt` | `application/SqlSafetyHalt.java` |
| `QueryResultScorer` | `application/QueryResultScorer.java` |
| `SchemaRegistry` | `application/SchemaRegistry.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSchemaStep` 10 s, `translateAndRunStep` 90 s, `scoreStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 90 s on `translateAndRunStep` accommodates LLM latency plus Postgres round-trip (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; `SchemaRegistryConsumer` is idempotent because `QueryEntity.attachSchema` is event-version-guarded — a second schema attach on an already-attached query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"agent-" + queryId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail vs. halt**: G1 (`SqlSafetyGuardrail`) fires first and allows one retry per rejected tool call. H1 (`SqlSafetyHalt`) fires on the same event as a backstop; if the agent produces an unsafe SQL after exhausting retries, the halt terminates the task and the entity moves to `HALTED`.
- **Scorer is synchronous and deterministic**: `QueryResultScorer` runs in-process inside `scoreStep`. No LLM call, no Postgres query — the same result always scores the same. This is the single-agent invariant.
- **Read-only database role**: `QueryExecutionTool` connects with a Postgres role that has SELECT-only permissions, provisioned by init.sql. Even without G1/H1, a write attempt would fail at the database level — the governance layers catch it earlier and surface a clean error.
