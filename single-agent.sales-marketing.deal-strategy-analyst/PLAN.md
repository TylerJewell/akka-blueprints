# PLAN — deal-strategy-analyst

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

  API[DealEndpoint]:::ep
  Entity[DealEntity]:::ese
  Sanitizer[DealContextSanitizer]:::cons
  WF[AnalysisWorkflow]:::wf
  Agent[DealStrategyAgent]:::agent
  Guard[RecommendationGuardrail]:::guard
  Scorer[RecommendationScorer]:::guard
  View[DealView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|DealSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|analyseStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|StrategyRecommendation| WF
  WF -->|recordRecommendation| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
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
  participant API as DealEndpoint
  participant E as DealEntity
  participant S as DealContextSanitizer
  participant W as AnalysisWorkflow
  participant A as DealStrategyAgent
  participant G as RecommendationGuardrail
  participant Sc as RecommendationScorer

  U->>API: POST /api/deals
  API->>E: submit(context)
  E-->>API: { dealId }
  E-.->>S: DealSubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(dealId)
  W->>E: poll getDeal
  E-->>W: sanitized.isPresent()
  W->>E: markAnalysing
  W->>A: runSingleTask(deal brief + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: StrategyRecommendation
  W->>E: recordRecommendation(recommendation)
  W->>Sc: score(recommendation, stakeholders)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `DealEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: ContextSanitized
  SANITIZED --> ANALYSING: AnalysisStarted
  ANALYSING --> RECOMMENDATION_RECORDED: RecommendationRecorded
  RECOMMENDATION_RECORDED --> EVALUATED: EvaluationScored
  ANALYSING --> FAILED: DealFailed (agent error)
  SUBMITTED --> FAILED: DealFailed (sanitizer error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  DealEntity ||--o{ DealSubmitted : emits
  DealEntity ||--o{ ContextSanitized : emits
  DealEntity ||--o{ AnalysisStarted : emits
  DealEntity ||--o{ RecommendationRecorded : emits
  DealEntity ||--o{ EvaluationScored : emits
  DealEntity ||--o{ DealFailed : emits
  DealView }o--|| DealEntity : projects
  DealContextSanitizer }o--|| DealEntity : subscribes
  AnalysisWorkflow }o--|| DealEntity : reads-and-writes
  DealStrategyAgent ||--o{ StrategyRecommendation : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DealEndpoint` | `api/DealEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DealEntity` | `application/DealEntity.java` (state in `domain/Deal.java`, events in `domain/DealEvent.java`) |
| `DealContextSanitizer` | `application/DealContextSanitizer.java` |
| `AnalysisWorkflow` | `application/AnalysisWorkflow.java` |
| `DealStrategyAgent` | `application/DealStrategyAgent.java` (tasks in `application/DealTasks.java`) |
| `RecommendationGuardrail` | `application/RecommendationGuardrail.java` |
| `RecommendationScorer` | `application/RecommendationScorer.java` |
| `DealView` | `application/DealView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `analyseStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AnalysisWorkflow::error)`. The 60 s on `analyseStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"analysis-" + dealId` as the workflow id; the `DealContextSanitizer` Consumer is allowed to redeliver `DealSubmitted` events because `DealEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized deal is a no-op.
- **One agent per deal**: the AutonomousAgent instance id is `"analyst-" + dealId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `RecommendationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `analyseStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `RecommendationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same recommendation always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
