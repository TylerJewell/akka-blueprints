# PLAN — youtube-analyst

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

  API[AnalysisEndpoint]:::ep
  Entity[ChannelEntity]:::ese
  Fetcher[MetricsFetcher]:::cons
  WF[AnalysisWorkflow]:::wf
  Agent[ChannelAnalystAgent]:::agent
  Guard[ReportGuardrail]:::guard
  Evaluator[AnalysisEvaluator]:::guard
  View[AnalysisView]:::view
  App[AppEndpoint]:::ep

  API -->|requestAnalysis| Entity
  Entity -.->|AnalysisRequested| Fetcher
  Fetcher -->|attachMetrics| Entity
  Fetcher -->|start workflow| WF
  WF -->|awaitMetricsStep poll| Entity
  WF -->|analyseStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|ChannelReport| WF
  WF -->|recordReport| Entity
  WF -->|evalStep score| Evaluator
  Evaluator -->|EvalResult| WF
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
  participant API as AnalysisEndpoint
  participant E as ChannelEntity
  participant F as MetricsFetcher
  participant W as AnalysisWorkflow
  participant A as ChannelAnalystAgent
  participant G as ReportGuardrail
  participant Ev as AnalysisEvaluator

  U->>API: POST /api/analyses
  API->>E: requestAnalysis(request)
  E-->>API: { analysisId }
  E-.->>F: AnalysisRequested
  F->>F: load fixture + compute metrics
  F->>E: attachMetrics
  F->>W: start(analysisId)
  W->>E: poll getAnalysis
  E-->>W: metrics.isPresent()
  W->>E: markAnalysing
  W->>A: runSingleTask(focusInstructions + metrics attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: ChannelReport
  W->>E: recordReport(report)
  W->>Ev: score(report, metrics)
  Ev-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `ChannelEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> METRICS_FETCHED: MetricsFetched
  METRICS_FETCHED --> ANALYSING: AnalysisStarted
  ANALYSING --> REPORT_RECORDED: ReportRecorded
  REPORT_RECORDED --> EVALUATED: EvaluationScored
  ANALYSING --> FAILED: AnalysisFailed (agent error)
  REQUESTED --> FAILED: AnalysisFailed (fetcher error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ChannelEntity ||--o{ AnalysisRequested : emits
  ChannelEntity ||--o{ MetricsFetched : emits
  ChannelEntity ||--o{ AnalysisStarted : emits
  ChannelEntity ||--o{ ReportRecorded : emits
  ChannelEntity ||--o{ EvaluationScored : emits
  ChannelEntity ||--o{ AnalysisFailed : emits
  AnalysisView }o--|| ChannelEntity : projects
  MetricsFetcher }o--|| ChannelEntity : subscribes
  AnalysisWorkflow }o--|| ChannelEntity : reads-and-writes
  ChannelAnalystAgent ||--o{ ChannelReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AnalysisEndpoint` | `api/AnalysisEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ChannelEntity` | `application/ChannelEntity.java` (state in `domain/ChannelAnalysis.java`, events in `domain/ChannelEvent.java`) |
| `MetricsFetcher` | `application/MetricsFetcher.java` |
| `AnalysisWorkflow` | `application/AnalysisWorkflow.java` |
| `ChannelAnalystAgent` | `application/ChannelAnalystAgent.java` (tasks in `application/AnalysisTasks.java`) |
| `ReportGuardrail` | `application/ReportGuardrail.java` |
| `AnalysisEvaluator` | `application/AnalysisEvaluator.java` |
| `AnalysisView` | `application/AnalysisView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitMetricsStep` 15 s, `analyseStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AnalysisWorkflow::error)`. The 60 s on `analyseStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"analysis-" + analysisId` as the workflow id; the `MetricsFetcher` Consumer is allowed to redeliver `AnalysisRequested` events because `ChannelEntity.attachMetrics` is event-version-guarded — a second fetch attempt against an already-fetched analysis is a no-op.
- **One agent per analysis**: the AutonomousAgent instance id is `"analyst-" + analysisId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReportGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `analyseStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `AnalysisEvaluator` runs in-process inside `evalStep`. No LLM call, no external service — the same report always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
