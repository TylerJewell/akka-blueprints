# PLAN — durable-weather-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef act fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[JobEndpoint]:::ep
  App[AppEndpoint]:::ep
  Entity[WeatherJobEntity]:::ese
  WF[AgentWorkflow]:::wf
  Agent[WeatherAgent]:::agent
  Activity[WeatherToolActivity]:::act
  Guard[ToolCallGuardrail]:::guard
  View[JobView]:::view

  API -->|queue| Entity
  API -->|triggerAgent / callAgent| WF
  WF -->|startAgentStep: start| Entity
  WF -->|runAgentStep: runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|pass / reject| Activity
  Activity -->|tool result| Agent
  Agent -->|WeatherReport| WF
  WF -->|finalizeStep: complete| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (trigger_agent happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as JobEndpoint
  participant E as WeatherJobEntity
  participant W as AgentWorkflow
  participant A as WeatherAgent
  participant G as ToolCallGuardrail
  participant T as WeatherToolActivity

  U->>API: POST /api/jobs (mode=TRIGGER_AGENT)
  API->>E: queue(query)
  E-->>API: { jobId }
  API->>W: triggerAgent(jobId)
  W->>E: start()
  W->>A: runSingleTask(queryText)
  A->>G: before-tool-call(get_current_weather, {city})
  G-->>A: pass
  A->>T: executeGetCurrentWeather(city)
  T-->>A: WeatherConditions fixture
  A->>G: before-tool-call(get_forecast, {city, days=3})
  G-->>A: pass
  A->>T: executeGetForecast(city, 3)
  T-->>A: List<ForecastDay>
  A->>G: before-tool-call(get_weather_alerts, {city, bbox})
  G-->>A: pass
  A->>T: executeGetWeatherAlerts(city, bbox)
  T-->>A: alerts list
  A-->>W: WeatherReport
  W->>E: complete(report)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `WeatherJobEntity`

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> RUNNING: JobStarted
  RUNNING --> COMPLETED: JobCompleted
  RUNNING --> FAILED: JobFailed (agent error or timeout)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  WeatherJobEntity ||--o{ JobQueued : emits
  WeatherJobEntity ||--o{ JobStarted : emits
  WeatherJobEntity ||--o{ JobCompleted : emits
  WeatherJobEntity ||--o{ JobFailed : emits
  JobView }o--|| WeatherJobEntity : projects
  AgentWorkflow }o--|| WeatherJobEntity : reads-and-writes
  WeatherAgent ||--o{ WeatherReport : returns
  WeatherAgent ||--o{ ToolCallRecord : accumulates
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `JobEndpoint` | `api/JobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `WeatherJobEntity` | `application/WeatherJobEntity.java` (state in `domain/WeatherJob.java`, events in `domain/WeatherJobEvent.java`) |
| `AgentWorkflow` | `application/AgentWorkflow.java` (inner `WeatherToolActivity`) |
| `WeatherAgent` | `application/WeatherAgent.java` (tasks in `application/WeatherTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `JobView` | `application/JobView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `startAgentStep` 10 s, `runAgentStep` 120 s, `finalizeStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AgentWorkflow::error)`. The 120 s on `runAgentStep` accommodates multiple sequential tool calls plus LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"agent-" + jobId` as the workflow id; `WeatherJobEntity.start` is guarded — a second `JobStarted` event on an already-running job is a no-op.
- **One agent per job**: the AutonomousAgent instance id is `"weather-" + jobId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(4)` accommodates guardrail-blocked tool calls that consume an iteration.
- **Guardrail-driven retry**: when `ToolCallGuardrail` rejects a tool call, the rejection is returned as `{ tool, argument, reason }` to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations exhaust the budget, the workflow's `runAgentStep` fails over to `error` and the entity transitions to `FAILED`.
- **call_agent composition**: a parent workflow step calls `asyncCall(AgentWorkflow::callAgent, childJobId)`. The parent step suspends until the child workflow's `done` transition fires. Both the parent and child job IDs are tracked independently in `WeatherJobEntity`; the UI shows them as separate cards.
- **Activity durability**: each `WeatherToolActivity` method is an Akka workflow activity, so if the process crashes after the HTTP call but before the agent receives the response, the activity replays without re-issuing the HTTP call (the result is stored in the workflow journal).
- **No saga / no compensation**: tool calls are idempotent fixtures in dev mode. In production, a deployer would replace the fixture calls with real HTTP and decide whether the tool is idempotent; if not, a deduplication key should be passed to the activity.
