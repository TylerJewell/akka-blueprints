# PLAN — structured-mcp-tool

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef mcp fill:#0e1a0e,stroke:#22d3ee,color:#22d3ee;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[WeatherQueryWorkflow]:::wf
  Agent[WeatherQueryAgent]:::agent
  Guard[McpToolResultGuardrail]:::guard
  MCP[InlineMcpServer]:::mcp
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|runStep runSingleTask| Agent
  Agent -->|tool call get_weather_info| MCP
  MCP -->|WeatherPayload| Agent
  Agent -.->|after-tool-call| Guard
  Guard -.->|pass or reject| Agent
  Agent -->|WeatherSummary| WF
  WF -->|recordSummary| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant W as WeatherQueryWorkflow
  participant A as WeatherQueryAgent
  participant G as McpToolResultGuardrail
  participant MCP as InlineMcpServer

  U->>API: POST /api/queries
  API->>E: submit(request)
  E-->>API: { queryId }
  API->>W: start(queryId)
  W->>E: markRunning
  W->>A: runSingleTask(location, unit)
  A->>MCP: get_weather_info(location, unit)
  MCP-->>A: WeatherPayload
  A->>G: after-tool-call(payload)
  G-->>A: accept
  A-->>W: WeatherSummary
  W->>E: recordSummary(summary)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RUNNING: QueryStarted
  RUNNING --> COMPLETED: SummaryRecorded
  RUNNING --> FAILED: QueryFailed (agent error or guardrail-exhaustion)
  SUBMITTED --> FAILED: QueryFailed (workflow start error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuerySubmitted : emits
  QueryEntity ||--o{ QueryStarted : emits
  QueryEntity ||--o{ SummaryRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  WeatherQueryWorkflow }o--|| QueryEntity : reads-and-writes
  WeatherQueryAgent ||--o{ WeatherSummary : returns
  InlineMcpServer ||--o{ WeatherPayload : exposes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `WeatherQueryWorkflow` | `application/WeatherQueryWorkflow.java` |
| `WeatherQueryAgent` | `application/WeatherQueryAgent.java` (tasks in `application/WeatherQueryTasks.java`) |
| `McpToolResultGuardrail` | `application/McpToolResultGuardrail.java` |
| `InlineMcpServer` | `application/InlineMcpServer.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `runStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(WeatherQueryWorkflow::error)`. The 60 s on `runStep` accommodates LLM latency and retry tool calls (Lesson 4).
- **Idempotency**: every workflow uses `"wq-" + queryId` as the workflow id. If the endpoint receives a duplicate POST for the same `queryId`, the entity's `submit` command is idempotent — it only emits `QuerySubmitted` if the entity is in its initial empty state; subsequent calls are no-ops.
- **One agent per query**: the AutonomousAgent instance id is `"query-agent-" + queryId`, giving each query its own conversation context. `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered tool-call retries at 3.
- **Guardrail-driven retry**: when `McpToolResultGuardrail` rejects a tool response, the rejection is returned to the agent loop as a structured error. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `runStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: the only external call is to the in-process `InlineMcpServer`. Nothing external requires rollback.
- **InlineMcpServer is stateless**: each `get_weather_info` call is independent. Seeded-location data is constant; all other locations derive from `location.hashCode()`. Determinism makes J2 and J3 reproducible.
