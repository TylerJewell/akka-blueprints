# PLAN — earth-engine-geospatial

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
  Entity[AnalysisEntity]:::ese
  Fetcher[DatasetFetcher]:::cons
  WF[AnalysisWorkflow]:::wf
  Agent[GeospatialAnalystAgent]:::agent
  Guard[ReportGuardrail]:::guard
  Scorer[EvaluationScorer]:::guard
  View[AnalysisView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|QuerySubmitted| Fetcher
  Fetcher -->|validate bounds + assemble dataset| Entity
  Fetcher -->|start workflow| WF
  WF -->|awaitDatasetStep poll| Entity
  WF -->|analysisStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|AnalysisReport| WF
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
  participant API as AnalysisEndpoint
  participant E as AnalysisEntity
  participant F as DatasetFetcher
  participant W as AnalysisWorkflow
  participant A as GeospatialAnalystAgent
  participant G as ReportGuardrail
  participant Sc as EvaluationScorer

  U->>API: POST /api/analyses
  API->>E: submit(query)
  E-->>API: { analysisId }
  E-.->>F: QuerySubmitted
  F->>F: validate bounds area
  F->>F: assemble DatasetSnapshot
  F->>E: attachDataset
  F->>W: start(analysisId)
  W->>E: poll getAnalysis
  E-->>W: dataset.isPresent()
  W->>E: markAnalysing
  W->>A: runSingleTask(indicators + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: AnalysisReport
  W->>E: recordReport(report)
  W->>Sc: score(report, indicators)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `AnalysisEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> DATASET_READY: DatasetReady
  DATASET_READY --> ANALYSING: AnalysisStarted
  ANALYSING --> REPORT_RECORDED: ReportRecorded
  REPORT_RECORDED --> EVALUATED: EvaluationScored
  SUBMITTED --> FAILED: AnalysisFailed (bounds exceeded)
  ANALYSING --> FAILED: AnalysisFailed (agent error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  AnalysisEntity ||--o{ QuerySubmitted : emits
  AnalysisEntity ||--o{ DatasetReady : emits
  AnalysisEntity ||--o{ AnalysisStarted : emits
  AnalysisEntity ||--o{ ReportRecorded : emits
  AnalysisEntity ||--o{ EvaluationScored : emits
  AnalysisEntity ||--o{ AnalysisFailed : emits
  AnalysisView }o--|| AnalysisEntity : projects
  DatasetFetcher }o--|| AnalysisEntity : subscribes
  AnalysisWorkflow }o--|| AnalysisEntity : reads-and-writes
  GeospatialAnalystAgent ||--o{ AnalysisReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AnalysisEndpoint` | `api/AnalysisEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AnalysisEntity` | `application/AnalysisEntity.java` (state in `domain/Analysis.java`, events in `domain/AnalysisEvent.java`) |
| `DatasetFetcher` | `application/DatasetFetcher.java` |
| `AnalysisWorkflow` | `application/AnalysisWorkflow.java` |
| `GeospatialAnalystAgent` | `application/GeospatialAnalystAgent.java` (tasks in `application/AnalysisTasks.java`) |
| `ReportGuardrail` | `application/ReportGuardrail.java` |
| `EvaluationScorer` | `application/EvaluationScorer.java` |
| `AnalysisView` | `application/AnalysisView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitDatasetStep` 15 s, `analysisStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AnalysisWorkflow::error)`. The 60 s on `analysisStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"analysis-" + analysisId` as the workflow id; `DatasetFetcher` is allowed to redeliver `QuerySubmitted` events because `AnalysisEntity.attachDataset` is event-version-guarded — a second dataset attach against an already-ready analysis is a no-op.
- **One agent per analysis**: the AutonomousAgent instance id is `"analyst-" + analysisId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReportGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. Each rejection counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `analysisStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `EvaluationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same report always scores the same.
- **Bounds check is pre-LLM**: `DatasetFetcher` rejects oversized bounding boxes before assembling any data and before starting the workflow. The entity transitions to `FAILED` directly from `SUBMITTED`, bypassing all downstream steps.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. Nothing external requires rollback.
