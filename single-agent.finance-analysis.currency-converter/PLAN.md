# PLAN — currency-agent

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

  API[ConversionEndpoint]:::ep
  Entity[ConversionEntity]:::ese
  Fetcher[RateFetcher]:::cons
  WF[ConversionWorkflow]:::wf
  Agent[ExchangeRateAgent]:::agent
  Guard[ResultGuardrail]:::guard
  Scorer[FreshnessScorer]:::guard
  View[ConversionView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|ConversionRequested| Fetcher
  Fetcher -->|attachRate| Entity
  Fetcher -->|start workflow| WF
  WF -->|awaitRateStep poll| Entity
  WF -->|convertStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|ConversionResult| WF
  WF -->|recordResult| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|FreshnessEval| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ConversionEndpoint
  participant E as ConversionEntity
  participant F as RateFetcher
  participant W as ConversionWorkflow
  participant A as ExchangeRateAgent
  participant G as ResultGuardrail
  participant Sc as FreshnessScorer

  U->>API: POST /api/conversions
  API->>E: submit(request)
  E-->>API: { conversionId }
  E-.->>F: ConversionRequested
  F->>F: load rate snapshot
  F->>E: attachRate(rateSnapshot)
  F->>W: start(conversionId)
  W->>E: poll getConversion
  E-->>W: rateSnapshot.isPresent()
  W->>E: markConverting
  W->>A: runSingleTask(params + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: ConversionResult
  W->>E: recordResult(result)
  W->>Sc: score(result, rateSnapshot)
  Sc-->>W: FreshnessEval
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `ConversionEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> RATE_ATTACHED: RateAttached
  RATE_ATTACHED --> CONVERTING: ConversionStarted
  CONVERTING --> RESULT_RECORDED: ResultRecorded
  RESULT_RECORDED --> EVALUATED: FreshnessScored
  CONVERTING --> FAILED: ConversionFailed (agent error)
  REQUESTED --> FAILED: ConversionFailed (fetcher error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversionEntity ||--o{ ConversionRequested : emits
  ConversionEntity ||--o{ RateAttached : emits
  ConversionEntity ||--o{ ConversionStarted : emits
  ConversionEntity ||--o{ ResultRecorded : emits
  ConversionEntity ||--o{ FreshnessScored : emits
  ConversionEntity ||--o{ ConversionFailed : emits
  ConversionView }o--|| ConversionEntity : projects
  RateFetcher }o--|| ConversionEntity : subscribes
  ConversionWorkflow }o--|| ConversionEntity : reads-and-writes
  ExchangeRateAgent ||--o{ ConversionResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversionEndpoint` | `api/ConversionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversionEntity` | `application/ConversionEntity.java` (state in `domain/Conversion.java`, events in `domain/ConversionEvent.java`) |
| `RateFetcher` | `application/RateFetcher.java` |
| `ConversionWorkflow` | `application/ConversionWorkflow.java` |
| `ExchangeRateAgent` | `application/ExchangeRateAgent.java` (tasks in `application/ConversionTasks.java`) |
| `ResultGuardrail` | `application/ResultGuardrail.java` |
| `FreshnessScorer` | `application/FreshnessScorer.java` |
| `ConversionView` | `application/ConversionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitRateStep` 15 s, `convertStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ConversionWorkflow::error)`. The 60 s on `convertStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"conv-" + conversionId` as the workflow id; the `RateFetcher` Consumer is allowed to redeliver `ConversionRequested` events because `ConversionEntity.attachRate` is event-version-guarded — a second rate-attach attempt on an already rate-attached conversion is a no-op.
- **One agent per conversion**: the AutonomousAgent instance id is `"agent-" + conversionId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ResultGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `convertStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `FreshnessScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same rate snapshot and result always score the same. This preserves the single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
