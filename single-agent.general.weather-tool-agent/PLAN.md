# PLAN — weather-agent

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
  classDef tools fill:#0e1e20,stroke:#38b2ac,color:#38b2ac;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[QueryWorkflow]:::wf
  Agent[WeatherAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  Tools[WeatherToolProvider]:::tools
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|processStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|pass / reject| Agent
  Agent -->|geocode / getWeather| Tools
  Tools -->|GeocodingResult / conditions| Agent
  Agent -->|WeatherAnswer| WF
  WF -->|recordAnswer| Entity
  WF -->|error| Entity
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
  participant G as ToolCallGuardrail
  participant T as WeatherToolProvider

  U->>API: POST /api/queries
  API->>E: submit(question)
  E-->>API: { questionId }
  API->>W: start(questionId)
  W->>E: startProcessing
  W->>A: runSingleTask(questionText)
  A->>G: before-tool-call(geocode, {location:"Paris"})
  G-->>A: accept
  A->>T: geocode("Paris")
  T-->>A: GeocodingResult{lat:48.85, lon:2.35}
  A->>G: before-tool-call(getWeather, {lat,lon,units,forecastDays})
  G-->>A: accept
  A->>T: getWeather(48.85, 2.35, "metric", 1)
  T-->>A: CurrentConditions{...}
  A-->>W: WeatherAnswer
  W->>E: recordAnswer(answer)
  E-.->>U: SSE event(ANSWERED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PROCESSING: ProcessingStarted
  PROCESSING --> ANSWERED: AnswerRecorded
  PROCESSING --> FAILED: QueryFailed (agent error or geocoding failure)
  ANSWERED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QuestionSubmitted : emits
  QueryEntity ||--o{ ProcessingStarted : emits
  QueryEntity ||--o{ ToolCallRecorded : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  WeatherAgent ||--o{ WeatherAnswer : returns
  WeatherAgent }o--|| WeatherToolProvider : calls
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `WeatherAgent` | `application/WeatherAgent.java` (tasks in `application/WeatherTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `WeatherToolProvider` | `application/WeatherToolProvider.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `processStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `processStep` accommodates up to two tool-call round trips plus LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"weather-" + questionId` as the agent instance id; the workflow id is `"wf-" + questionId`. If `QueryEndpoint` retries a start call, the workflow resumes from its current step rather than restarting.
- **One agent per query**: the AutonomousAgent instance id is `"weather-" + questionId`, giving each task its own conversation context. `maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail-driven retry**: when `ToolCallGuardrail` rejects a parameter set, the rejection is returned as a tool-call error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow's `processStep` fails over to `error` and the entity transitions to `FAILED`.
- **Tool stubs are synchronous**: `WeatherToolProvider` stub methods return immediately — no I/O, no thread pools. A real implementation would wrap an HTTP client inside a `CompletableFuture`, but that is deferred to deployer configuration.
- **No saga / no compensation**: the workflow has two steps (processStep → error path only). There is nothing external to roll back — all side effects are entity events.
