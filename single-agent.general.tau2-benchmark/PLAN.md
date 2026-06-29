# PLAN — tau2-benchmark-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef scorer fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  API[BenchmarkEndpoint]:::ep
  Entity[BenchmarkRunEntity]:::ese
  WF[BenchmarkRunWorkflow]:::wf
  Agent[BenchmarkAgent]:::agent
  Scorer[TaskScorer]:::scorer
  View[BenchmarkView]:::view
  App[AppEndpoint]:::ep

  API -->|submit + start workflow| Entity
  API -->|start workflow| WF
  WF -->|executeStep runSingleTask| Agent
  Agent -->|TaskResult| WF
  WF -->|recordResult| Entity
  WF -->|scoreStep score| Scorer
  Scorer -->|PerformanceScore| WF
  WF -->|recordScore| Entity
  Entity -.->|projects| View
  API -->|list/SSE/aggregate| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BenchmarkEndpoint
  participant E as BenchmarkRunEntity
  participant W as BenchmarkRunWorkflow
  participant A as BenchmarkAgent
  participant Sc as TaskScorer

  U->>API: POST /api/runs
  API->>E: submit(request)
  E-->>API: { runId }
  API->>W: start(runId)
  W->>E: markExecuting
  W->>A: runSingleTask(description + task.json attachment)
  A-->>W: TaskResult
  W->>E: recordResult(result)
  W->>Sc: score(result, task)
  Sc-->>W: PerformanceScore
  W->>E: recordScore(score)
  E-.->>U: SSE event(SCORED)
```

## State machine — `BenchmarkRunEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> EXECUTING: ExecutionStarted
  EXECUTING --> RESULT_RECORDED: ResultRecorded
  RESULT_RECORDED --> SCORED: RunScored
  EXECUTING --> FAILED: RunFailed (agent timeout or error)
  SUBMITTED --> FAILED: RunFailed (workflow start error)
  SCORED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BenchmarkRunEntity ||--o{ RunSubmitted : emits
  BenchmarkRunEntity ||--o{ ExecutionStarted : emits
  BenchmarkRunEntity ||--o{ ResultRecorded : emits
  BenchmarkRunEntity ||--o{ RunScored : emits
  BenchmarkRunEntity ||--o{ RunFailed : emits
  BenchmarkView }o--|| BenchmarkRunEntity : projects
  BenchmarkRunWorkflow }o--|| BenchmarkRunEntity : reads-and-writes
  BenchmarkAgent ||--o{ TaskResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BenchmarkEndpoint` | `api/BenchmarkEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BenchmarkRunEntity` | `application/BenchmarkRunEntity.java` (state in `domain/BenchmarkRun.java`, events in `domain/BenchmarkRunEvent.java`) |
| `BenchmarkRunWorkflow` | `application/BenchmarkRunWorkflow.java` |
| `BenchmarkAgent` | `application/BenchmarkAgent.java` (tasks in `application/BenchmarkTasks.java`) |
| `TaskScorer` | `application/TaskScorer.java` |
| `BenchmarkView` | `application/BenchmarkView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `executeStep` 60 s, `scoreStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BenchmarkRunWorkflow::error)`. The 60 s on `executeStep` accommodates LLM latency for multi-step reasoning tasks (Lesson 4).
- **Idempotency**: every workflow uses `"run-" + runId` as the workflow id; duplicate `POST /api/runs` calls with the same `runId` are rejected by `BenchmarkRunEntity.submit` once the entity is in a non-initial state.
- **One agent per run**: the AutonomousAgent instance id is `"agent-" + runId`, giving each task its own conversation context. `maxIterationsPerTask(3)` caps retries at 3.
- **Scorer is synchronous and deterministic**: `TaskScorer` runs in-process inside `scoreStep`. No LLM call, no external service — the same result always scores the same. This is the single-agent guarantee.
- **Aggregate computed on read**: pass rate, mean score, and per-category counts are derived from the full run list on `GET /api/runs/aggregate`. No materialised aggregate state — the list is small for a benchmark run.
- **No saga / no compensation**: each step is a single forward write. There is nothing external to roll back.
