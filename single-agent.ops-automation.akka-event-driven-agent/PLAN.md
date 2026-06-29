# PLAN — ambient-durable-agent-pubsub

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
  classDef san fill:#0e1a2a,stroke:#60a5fa,color:#60a5fa;

  API[RequestEndpoint]:::ep
  Entity[RequestEntity]:::ese
  Consumer[TopicConsumer]:::cons
  WF[WeatherWorkflow]:::wf
  Agent[WeatherAgent]:::agent
  Guard[PayloadGuardrail]:::guard
  Sanitizer[RequestSanitizer]:::san
  View[RequestView]:::view
  App[AppEndpoint]:::ep
  Topic([weather.requests topic]):::cons

  API -->|publish message| Topic
  Topic -.->|message| Consumer
  Consumer -->|receive| Entity
  Consumer -->|start workflow| WF
  WF -->|validateStep call| Guard
  Guard -.->|ValidationResult| WF
  WF -->|markValidated| Entity
  WF -->|sanitizeStep call| Sanitizer
  Sanitizer -.->|SanitizedPayload| WF
  WF -->|attachSanitized| Entity
  WF -->|forecastStep runSingleTask| Agent
  Agent -->|WeatherReport| WF
  WF -->|recordReport| Entity
  WF -->|complete| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as RequestEndpoint
  participant T as Topic broker
  participant C as TopicConsumer
  participant E as RequestEntity
  participant W as WeatherWorkflow
  participant G as PayloadGuardrail
  participant S as RequestSanitizer
  participant A as WeatherAgent

  U->>API: POST /api/requests/publish
  API->>T: publish(WeatherRequestPayload)
  API-->>U: { requestId }
  T-.->>C: message delivery
  C->>E: receive(payload)
  C->>W: start("forecast-" + requestId)
  W->>G: validate(rawPayload)
  G-->>W: ValidationResult{valid=true}
  W->>E: markValidated()
  W->>S: sanitize(rawPayload)
  S-->>W: SanitizedPayload
  W->>E: attachSanitized(sanitized)
  W->>E: markForecasting()
  W->>A: runSingleTask(context + attachment)
  A-->>W: WeatherReport
  W->>E: recordReport(report)
  W->>E: complete()
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `RequestEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> VALIDATED: PayloadValidated (valid)
  RECEIVED --> FAILED: PayloadValidated (invalid) / RequestFailed
  VALIDATED --> SANITIZED: PayloadSanitized
  SANITIZED --> FORECASTING: ForecastingStarted
  FORECASTING --> COMPLETED: ReportRecorded
  FORECASTING --> FAILED: RequestFailed (agent error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RequestEntity ||--o{ MessageReceived : emits
  RequestEntity ||--o{ PayloadValidated : emits
  RequestEntity ||--o{ PayloadSanitized : emits
  RequestEntity ||--o{ ForecastingStarted : emits
  RequestEntity ||--o{ ReportRecorded : emits
  RequestEntity ||--o{ RequestFailed : emits
  RequestView }o--|| RequestEntity : projects
  TopicConsumer }o--|| RequestEntity : receives-and-starts
  WeatherWorkflow }o--|| RequestEntity : reads-and-writes
  WeatherAgent ||--o{ WeatherReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RequestEndpoint` | `api/RequestEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `RequestEntity` | `application/RequestEntity.java` (state in `domain/Request.java`, events in `domain/RequestEvent.java`) |
| `TopicConsumer` | `application/TopicConsumer.java` |
| `WeatherWorkflow` | `application/WeatherWorkflow.java` |
| `WeatherAgent` | `application/WeatherAgent.java` (tasks in `application/ForecastTasks.java`) |
| `PayloadGuardrail` | `application/PayloadGuardrail.java` |
| `RequestSanitizer` | `application/RequestSanitizer.java` |
| `RequestView` | `application/RequestView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `validateStep` 5 s, `sanitizeStep` 5 s, `forecastStep` 60 s, `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(WeatherWorkflow::error)`. The 60 s on `forecastStep` accommodates LLM latency (Lesson 4).
- **Per-message isolation**: each topic message is processed by an independent `WeatherWorkflow` instance keyed to `"forecast-" + requestId`. There is no shared state between concurrent workflow executions.
- **Idempotency**: `TopicConsumer` may redeliver the same message if the broker retries. `RequestEntity.receive()` is event-version-guarded — a second `MessageReceived` event for an already-received `requestId` is a no-op.
- **Validation-first invariant**: `PayloadGuardrail.validate()` runs as the very first workflow step. If it returns `valid == false`, the workflow transitions the entity to FAILED and calls `thenEnd()` without ever reaching `forecastStep`. The agent task is never started.
- **Sanitize-before-agent invariant**: `forecastStep` receives only the `SanitizedPayload` produced in `sanitizeStep`. The raw `WeatherRequestPayload` is never passed to the agent. This is structurally enforced — `forecastStep` does not have access to the raw payload in its input type.
- **One agent per request**: the AutonomousAgent instance id is `"weather-" + requestId`, giving each task its own conversation context. `maxIterationsPerTask(2)` caps the agent loop.
- **No saga / no compensation**: every step either validates, sanitizes (pure function), calls the agent (single task), or writes to the entity. Nothing external to roll back.
