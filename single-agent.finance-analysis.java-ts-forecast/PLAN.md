# PLAN — time-series-forecasting (java)

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
  classDef eval fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ForecastEndpoint]:::ep
  Entity[ForecastEntity]:::ese
  Validator[SeriesValidator]:::cons
  WF[ForecastWorkflow]:::wf
  Agent[ForecastingAgent]:::agent
  Guard[ForecastGuardrail]:::eval
  Drift[DriftEvaluator]:::eval
  View[ForecastView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|SeriesSubmitted| Validator
  Validator -->|attachValidated| Entity
  Validator -->|start workflow| WF
  WF -->|awaitValidatedStep poll| Entity
  WF -->|forecastStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|ForecastResult| WF
  WF -->|recordForecast| Entity
  WF -->|driftEvalStep score| Drift
  Drift -->|DriftEval| WF
  WF -->|recordDriftEval| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ForecastEndpoint
  participant E as ForecastEntity
  participant V as SeriesValidator
  participant W as ForecastWorkflow
  participant A as ForecastingAgent
  participant G as ForecastGuardrail
  participant D as DriftEvaluator

  U->>API: POST /api/forecasts
  API->>E: submit(submission)
  E-->>API: { forecastId }
  E-.->>V: SeriesSubmitted
  V->>V: validate quality + stationarity
  V->>E: attachValidated
  V->>W: start(forecastId)
  W->>E: poll getForecast
  E-->>W: quality.isPresent()
  W->>E: markForecasting
  W->>A: runSingleTask(config + series.csv attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: ForecastResult
  W->>E: recordForecast(result)
  W->>D: evaluate(result, historicalData)
  D-->>W: DriftEval
  W->>E: recordDriftEval(driftEval)
  E-.->>U: SSE event(DRIFT_EVALUATED)
```

## State machine — `ForecastEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> VALIDATED: SeriesValidated
  VALIDATED --> FORECASTING: ForecastStarted
  FORECASTING --> FORECAST_COMPLETED: ForecastCompleted
  FORECAST_COMPLETED --> DRIFT_EVALUATED: DriftEvaluated
  FORECASTING --> FAILED: ForecastFailed (agent error)
  SUBMITTED --> FAILED: ForecastFailed (validator error)
  DRIFT_EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ForecastEntity ||--o{ SeriesSubmitted : emits
  ForecastEntity ||--o{ SeriesValidated : emits
  ForecastEntity ||--o{ ForecastStarted : emits
  ForecastEntity ||--o{ ForecastCompleted : emits
  ForecastEntity ||--o{ DriftEvaluated : emits
  ForecastEntity ||--o{ ForecastFailed : emits
  ForecastView }o--|| ForecastEntity : projects
  SeriesValidator }o--|| ForecastEntity : subscribes
  ForecastWorkflow }o--|| ForecastEntity : reads-and-writes
  ForecastingAgent ||--o{ ForecastResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ForecastEndpoint` | `api/ForecastEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ForecastEntity` | `application/ForecastEntity.java` (state in `domain/Forecast.java`, events in `domain/ForecastEvent.java`) |
| `SeriesValidator` | `application/SeriesValidator.java` |
| `ForecastWorkflow` | `application/ForecastWorkflow.java` |
| `ForecastingAgent` | `application/ForecastingAgent.java` (tasks in `application/ForecastTasks.java`) |
| `ForecastGuardrail` | `application/ForecastGuardrail.java` |
| `DriftEvaluator` | `application/DriftEvaluator.java` |
| `ForecastView` | `application/ForecastView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitValidatedStep` 15 s, `forecastStep` 60 s, `driftEvalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ForecastWorkflow::error)`. The 60 s on `forecastStep` accommodates LLM latency on long series (Lesson 4).
- **Idempotency**: every workflow uses `"forecast-" + forecastId` as the workflow id; the `SeriesValidator` Consumer is allowed to redeliver `SeriesSubmitted` events because `ForecastEntity.attachValidated` is event-version-guarded — a second validate attempt against an already-validated forecast is a no-op.
- **One agent per forecast**: the AutonomousAgent instance id is `"forecaster-" + forecastId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ForecastGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `forecastStep` fails over to `error` and the entity transitions to `FAILED`.
- **Drift eval is synchronous and deterministic**: `DriftEvaluator` runs in-process inside `driftEvalStep`. No LLM call, no external service — the same series and result always score the same drift status. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
