# PLAN — any-llm-tool-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[QueryWorkflow]:::wf
  Agent[WeatherAgent]:::agent
  Guard[WeatherToolGuardrail]:::guard
  Tools[WeatherTools]:::tool
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|invokeStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|get_weather| Tools
  Tools -->|WeatherData| Agent
  Agent -->|WeatherReport| WF
  WF -->|recordReport| Entity
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
  participant A as WeatherAgent
  participant G as WeatherToolGuardrail
  participant T as WeatherTools

  U->>API: POST /api/queries
  API->>E: submit(query)
  API->>W: start(queryId)
  E-->>API: { queryId }
  W->>E: markInvoking
  W->>A: runSingleTask(queryText)
  A->>G: before-tool-call(get_weather, location)
  G-->>A: accept
  A->>T: get_weather(location)
  T-->>A: WeatherData
  A-->>W: WeatherReport
  W->>E: recordReport(report)
  E-.->>U: SSE event(RESULT_RECORDED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> INVOKING: InvocationStarted
  INVOKING --> RESULT_RECORDED: ReportRecorded
  INVOKING --> FAILED: QueryFailed (agent error)
  SUBMITTED --> FAILED: QueryFailed (workflow error)
  RESULT_RECORDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ InvocationStarted : emits
  QueryEntity ||--o{ ReportRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  WeatherAgent ||--o{ WeatherReport : returns
  WeatherTools ||--o{ WeatherData : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `WeatherAgent` | `application/WeatherAgent.java` (tasks in `application/QueryTasks.java`) |
| `WeatherToolGuardrail` | `application/WeatherToolGuardrail.java` |
| `WeatherTools` | `application/WeatherTools.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `invokeStep` 60 s, `recordStep` 10 s, `error` 10 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `invokeStep` accommodates LLM latency across all supported backends (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id. The `QueryEndpoint` mints the UUID; calling `start(queryId)` a second time is a no-op because the workflow is already running.
- **One agent per query**: the AutonomousAgent instance id is `"weather-" + queryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries.
- **Guardrail-driven clarification**: when `WeatherToolGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The agent is expected to respond with a clarification report (e.g., "I could not determine a valid location from your query.") rather than retrying with the same input. The workflow records this clarification as the `WeatherReport.narrative` and transitions to `RESULT_RECORDED`.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
- **Backend selection**: `requestedBackend` is stored on `WeatherQuery` and forwarded to the workflow; the workflow passes it as a hint when constructing the `TaskDef`. The actual provider switch is handled by `application.conf` — the Java code does not branch on the backend value beyond logging it.
