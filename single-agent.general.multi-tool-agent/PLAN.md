# PLAN — multi-tool-agent

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

  API[SessionEndpoint]:::ep
  Entity[SessionEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[ToolCallingAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  Weather[WeatherTool]:::tool
  Currency[CurrencyTool]:::tool
  Unit[UnitTool]:::tool
  View[SessionView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|validateStep| Entity
  WF -->|dispatchStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|pass / reject| Agent
  Agent -->|WeatherTool| Weather
  Agent -->|CurrencyTool| Currency
  Agent -->|UnitTool| Unit
  Weather -->|WeatherResult| Agent
  Currency -->|CurrencyResult| Agent
  Unit -->|UnitResult| Agent
  Agent -->|ToolResponse| WF
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
  participant API as SessionEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant A as ToolCallingAgent
  participant G as ToolCallGuardrail
  participant Wt as WeatherTool
  participant Cx as CurrencyTool

  U->>API: POST /api/sessions
  API->>E: submit(request)
  API->>W: start(sessionId)
  E-->>API: { sessionId }
  W->>E: validateStep → startDispatch
  W->>A: runSingleTask(requestText)
  A->>G: before-tool-call(WeatherTool, {city:"Tokyo"})
  G-->>A: accept
  A->>Wt: lookup("Tokyo")
  Wt-->>A: WeatherResult
  A->>G: before-tool-call(CurrencyTool, {85,"USD","EUR"})
  G-->>A: accept
  A->>Cx: convert(85,"USD","EUR")
  Cx-->>A: CurrencyResult
  A-->>W: ToolResponse
  W->>E: recordAnswer(response)
  E-.->>U: SSE event(ANSWERED)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> DISPATCHING: DispatchStarted
  DISPATCHING --> ANSWERED: SessionAnswered
  DISPATCHING --> FAILED: SessionFailed (agent error / iteration budget)
  SUBMITTED --> FAILED: SessionFailed (validation error)
  ANSWERED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionSubmitted : emits
  SessionEntity ||--o{ DispatchStarted : emits
  SessionEntity ||--o{ ToolCallRecorded : emits
  SessionEntity ||--o{ SessionAnswered : emits
  SessionEntity ||--o{ SessionFailed : emits
  SessionView }o--|| SessionEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  ToolCallingAgent ||--o{ ToolResponse : returns
  ToolCallingAgent }o--|| WeatherTool : calls
  ToolCallingAgent }o--|| CurrencyTool : calls
  ToolCallingAgent }o--|| UnitTool : calls
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `ToolCallingAgent` | `application/ToolCallingAgent.java` (tasks in `application/SessionTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `WeatherTool` | `application/WeatherTool.java` |
| `CurrencyTool` | `application/CurrencyTool.java` |
| `UnitTool` | `application/UnitTool.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `validateStep` 5 s, `dispatchStep` 120 s, `summarizeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(SessionWorkflow::error)`. The 120 s on `dispatchStep` accommodates multiple sequential tool calls plus LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"session-" + sessionId` as the workflow id; `SessionEndpoint` guards against duplicate submits by checking entity state before starting the workflow.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`, giving each session its own conversation context. The agent's `capability(...).maxIterationsPerTask(5)` caps guardrail-triggered retries at 5.
- **Guardrail-driven reformulation**: when `ToolCallGuardrail` rejects a tool call, the rejection is returned as a structured `invalid-tool-call` error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 5 iterations fail to produce a valid tool call, the workflow's `dispatchStep` fails over to `error` and the entity transitions to `FAILED`.
- **Tool adapters are synchronous and in-process**: `WeatherTool`, `CurrencyTool`, and `UnitTool` return immediately from seeded data. No external HTTP. This is deliberate — the blueprint demonstrates the guardrail and agent pattern; a real deployer swaps the adapters for real HTTP clients.
- **No saga / no compensation**: every step is either pure validation, append-only event write, or a single-task agent call. There is nothing external to roll back.
