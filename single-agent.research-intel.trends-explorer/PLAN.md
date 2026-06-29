# PLAN — google-trends-agent

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

  API[TrendEndpoint]:::ep
  Entity[TrendRequestEntity]:::ese
  Fetcher[TrendDataFetcher]:::cons
  WF[TrendRequestWorkflow]:::wf
  Agent[TrendsExplorerAgent]:::agent
  Guard[ReportQualityGuardrail]:::guard
  Scorer[ReportEvaluationScorer]:::guard
  View[TrendReportView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|TrendQuerySubmitted| Fetcher
  Fetcher -->|attachTrendData| Entity
  Fetcher -->|start workflow| WF
  WF -->|awaitDataStep poll| Entity
  WF -->|synthesizeStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|TrendReport| WF
  WF -->|recordReport| Entity
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
  participant API as TrendEndpoint
  participant E as TrendRequestEntity
  participant F as TrendDataFetcher
  participant W as TrendRequestWorkflow
  participant A as TrendsExplorerAgent
  participant G as ReportQualityGuardrail
  participant Sc as ReportEvaluationScorer

  U->>API: POST /api/trends
  API->>E: submit(query)
  E-->>API: { requestId }
  E-.->>F: TrendQuerySubmitted
  F->>F: look up seeded dataset
  F->>E: attachTrendData
  F->>W: start(requestId)
  W->>E: poll getTrendRequest
  E-->>W: rawPayload.isPresent()
  W->>E: markSynthesizing
  W->>A: runSingleTask(params + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: TrendReport
  W->>E: recordReport(report)
  W->>Sc: score(report)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `TrendRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> DATA_READY: TrendDataReady
  DATA_READY --> SYNTHESIZING: SynthesisStarted
  SYNTHESIZING --> REPORT_READY: ReportRecorded
  REPORT_READY --> EVALUATED: EvaluationScored
  SYNTHESIZING --> FAILED: TrendRequestFailed (agent error)
  SUBMITTED --> FAILED: TrendRequestFailed (fetcher error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TrendRequestEntity ||--o{ TrendQuerySubmitted : emits
  TrendRequestEntity ||--o{ TrendDataReady : emits
  TrendRequestEntity ||--o{ SynthesisStarted : emits
  TrendRequestEntity ||--o{ ReportRecorded : emits
  TrendRequestEntity ||--o{ EvaluationScored : emits
  TrendRequestEntity ||--o{ TrendRequestFailed : emits
  TrendReportView }o--|| TrendRequestEntity : projects
  TrendDataFetcher }o--|| TrendRequestEntity : subscribes
  TrendRequestWorkflow }o--|| TrendRequestEntity : reads-and-writes
  TrendsExplorerAgent ||--o{ TrendReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TrendEndpoint` | `api/TrendEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TrendRequestEntity` | `application/TrendRequestEntity.java` (state in `domain/TrendRequest.java`, events in `domain/TrendRequestEvent.java`) |
| `TrendDataFetcher` | `application/TrendDataFetcher.java` |
| `TrendRequestWorkflow` | `application/TrendRequestWorkflow.java` |
| `TrendsExplorerAgent` | `application/TrendsExplorerAgent.java` (tasks in `application/TrendTasks.java`) |
| `ReportQualityGuardrail` | `application/ReportQualityGuardrail.java` |
| `ReportEvaluationScorer` | `application/ReportEvaluationScorer.java` |
| `TrendReportView` | `application/TrendReportView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitDataStep` 15 s, `synthesizeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TrendRequestWorkflow::error)`. The 60 s on `synthesizeStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"trend-" + requestId` as the workflow id; the `TrendDataFetcher` Consumer is allowed to redeliver `TrendQuerySubmitted` events because `TrendRequestEntity.attachTrendData` is event-version-guarded — a second data-attach on an already-data-ready request is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"explorer-" + requestId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReportQualityGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `synthesizeStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `ReportEvaluationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same report always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
