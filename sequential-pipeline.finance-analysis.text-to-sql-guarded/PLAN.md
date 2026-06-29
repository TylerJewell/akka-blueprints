# PLAN — text-to-sql-guarded

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef sanitizer fill:#1a0e2a,stroke:#c084fc,color:#c084fc;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[QueryPipelineWorkflow]:::wf
  Agent[SqlAgent]:::agent
  Parse[ParseTools]:::tool
  Query[QueryTools]:::tool
  Format[FormatTools]:::tool
  Guard[SqlGuardrail]:::guard
  Safety[SqlSafetyInspector]:::guard
  Sanitizer[PiiSanitizer]:::sanitizer
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|parseStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Parse
  Agent -->|invokes| Query
  Agent -->|invokes| Format
  Guard -->|recordGuardrailRejection| Entity
  WF -->|inspect sql| Safety
  Safety -->|SafetyHaltFired| Entity
  Agent -->|ParsedQuery| WF
  WF -->|sanitize rawResult| Sanitizer
  Sanitizer -->|SanitizedResult| WF
  Agent -->|RawResult / QueryReport| WF
  WF -->|recordParsed/QueryResult/Report| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant W as QueryPipelineWorkflow
  participant A as SqlAgent
  participant G as SqlGuardrail
  participant T as Tools (Parse/Query/Format)
  participant SI as SqlSafetyInspector
  participant PS as PiiSanitizer

  U->>API: POST /api/queries { question }
  API->>E: create(question)
  E-->>API: { queryId }
  API->>W: start(queryId, question)
  W->>E: startParse
  W->>A: runSingleTask(PARSE_QUESTION, question)
  A->>G: before-tool-call(resolveSchema, PARSE)
  G-->>A: accept (status PARSING)
  A->>T: resolveSchema + generateSql
  T-->>A: ParsedQuery
  A-->>W: ParsedQuery
  W->>SI: inspect(sql)
  SI-->>W: safe=true
  W->>E: recordParsed(parsedQuery)
  W->>A: runSingleTask(EXECUTE_QUERY, parsedQuery)
  A->>G: before-tool-call(executeSql, QUERY)
  G-->>A: accept (status QUERYING and parsedQuery present)
  A->>T: executeSql
  T-->>A: RawResult
  A-->>W: RawResult
  W->>E: recordQueryResult(rawResult)
  W->>PS: sanitize(rawResult)
  PS-->>W: SanitizedResult
  W->>A: runSingleTask(FORMAT_RESULTS, sanitizedResult)
  A->>G: before-tool-call(narrativeSummary, FORMAT)
  G-->>A: accept (status FORMATTING and rawResult present)
  A->>T: narrativeSummary + tableLayout
  T-->>A: QueryReport
  A-->>W: QueryReport
  W->>E: recordReport(report)
  E-.->>U: SSE event(FORMATTED)
```

## State machine — `QueryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PARSING: ParseStarted
  PARSING --> PARSED: QuestionParsed
  PARSING --> HALTED: SafetyHaltFired
  PARSED --> QUERYING: QueryStarted
  QUERYING --> QUERIED: QueryExecuted
  QUERIED --> FORMATTING: FormatStarted
  FORMATTING --> FORMATTED: ResultsFormatted
  PARSING --> FAILED: QueryFailed
  QUERYING --> FAILED: QueryFailed
  FORMATTING --> FAILED: QueryFailed
  FORMATTED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. SafetyHaltFired is terminal: the workflow ends immediately and no retry path exists for the same run. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ ParseStarted : emits
  QueryEntity ||--o{ QuestionParsed : emits
  QueryEntity ||--o{ SafetyHaltFired : emits
  QueryEntity ||--o{ QueryStarted : emits
  QueryEntity ||--o{ QueryExecuted : emits
  QueryEntity ||--o{ FormatStarted : emits
  QueryEntity ||--o{ ResultsFormatted : emits
  QueryEntity ||--o{ GuardrailRejected : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  QueryPipelineWorkflow }o--|| QueryEntity : reads-and-writes
  SqlAgent ||--o{ ParsedQuery : returns
  SqlAgent ||--o{ RawResult : returns
  SqlAgent ||--o{ QueryReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/QueryRecord.java`, events in `domain/QueryEvent.java`) |
| `QueryPipelineWorkflow` | `application/QueryPipelineWorkflow.java` |
| `SqlAgent` | `application/SqlAgent.java` (tasks in `application/SqlTasks.java`) |
| `ParseTools` | `application/ParseTools.java` |
| `QueryTools` | `application/QueryTools.java` |
| `FormatTools` | `application/FormatTools.java` |
| `SqlGuardrail` | `application/SqlGuardrail.java` |
| `SqlSafetyInspector` | `application/SqlSafetyInspector.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `parseStep` 60 s, `queryStep` 60 s, `formatStep` 60 s, `haltStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Safety halt is synchronous and non-retryable**: `SqlSafetyInspector` runs in-process immediately after the agent returns `ParsedQuery`, before any network call to the database. A halted workflow writes `SafetyHaltFired` and ends; no retry path. The user submits a new question.
- **PII sanitizer runs before the FORMAT agent task**: the `RawResult` is sanitized inside `formatStep` before `runSingleTask(FORMAT_RESULTS)` is called. The agent's FORMAT task context carries the sanitized rows only; the raw rows are never passed to the LLM.
- **Idempotency**: each workflow uses `"pipeline-" + queryId` as the workflow id; restart of the same queryId is rejected by the workflow runtime. The agent instance id is `"agent-" + queryId`.
- **One agent per query**: `SqlAgent` runs three tasks per query — PARSE, QUERY, FORMAT — each with `capability(...).maxIterationsPerTask(4)`.
- **Guardrail-driven retry**: when `SqlGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. If all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Task-boundary handoff is the dependency contract**: `parseStep` writes `QuestionParsed` BEFORE returning; `queryStep` reads the recorded `ParsedQuery` to build its task's instruction context; `formatStep` reads both `parsedQuery` and the sanitized result. The agent itself is stateless across phases.
