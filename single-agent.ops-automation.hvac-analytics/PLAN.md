# PLAN — hvac-analytics

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
  classDef scorer fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  Store[TelemetryStore]:::cons
  WF[QueryWorkflow]:::wf
  Agent[HvacAnalyticsAgent]:::agent
  Scorer[AnswerQualityScorer]:::scorer
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|initiate| Entity
  Entity -.->|QueryInitiated| Store
  Store -->|attachSnapshot| Entity
  Store -->|start workflow| WF
  WF -->|awaitSnapshotStep poll| Entity
  WF -->|analysisStep runSingleTask| Agent
  Agent -->|AnalyticsAnswer| WF
  WF -->|recordAnswer| Entity
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
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant St as TelemetryStore
  participant W as QueryWorkflow
  participant A as HvacAnalyticsAgent
  participant Sc as AnswerQualityScorer

  U->>API: POST /api/queries
  API->>E: initiate(request)
  E-->>API: { queryId }
  E-.->>St: QueryInitiated
  St->>St: assemble snapshot (strip serial ids)
  St->>E: attachSnapshot
  St->>W: start(queryId)
  W->>E: poll getQuery
  E-->>W: snapshot.isPresent()
  W->>E: markAnalysing
  W->>A: runSingleTask(question + attachment)
  A-->>W: AnalyticsAnswer
  W->>E: recordAnswer(answer)
  W->>Sc: score(answer)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
stateDiagram-v2
  [*] --> INITIATED
  INITIATED --> SNAPSHOT_READY: SnapshotAttached
  SNAPSHOT_READY --> ANALYSING: AnalysisStarted
  ANALYSING --> ANSWER_RECORDED: AnswerRecorded
  ANSWER_RECORDED --> EVALUATED: EvaluationScored
  ANALYSING --> FAILED: QueryFailed (agent error)
  INITIATED --> FAILED: QueryFailed (store error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  QueryEntity ||--o{ QueryInitiated : emits
  QueryEntity ||--o{ SnapshotAttached : emits
  QueryEntity ||--o{ AnalysisStarted : emits
  QueryEntity ||--o{ AnswerRecorded : emits
  QueryEntity ||--o{ EvaluationScored : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  TelemetryStore }o--|| QueryEntity : subscribes
  QueryWorkflow }o--|| QueryEntity : reads-and-writes
  HvacAnalyticsAgent ||--o{ AnalyticsAnswer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `TelemetryStore` | `application/TelemetryStore.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `HvacAnalyticsAgent` | `application/HvacAnalyticsAgent.java` (tasks in `application/QueryTasks.java`) |
| `AnswerQualityScorer` | `application/AnswerQualityScorer.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSnapshotStep` 15 s, `analysisStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(QueryWorkflow::error)`. The 60 s on `analysisStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"query-" + queryId` as the workflow id; the `TelemetryStore` Consumer is allowed to redeliver `QueryInitiated` events because `QueryEntity.attachSnapshot` is event-version-guarded — a second snapshot attempt against an already-snapshotted query is a no-op.
- **One agent per query**: the AutonomousAgent instance id is `"analyst-" + queryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps retries at 3.
- **Eval is synchronous and deterministic**: `AnswerQualityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
- **Telemetry isolation**: `TelemetryStore` strips equipment-level serial-number fields from every snapshot before calling `attachSnapshot`. The in-process data store retains the full fidelity records; the agent sees only zone-labelled metrics.
