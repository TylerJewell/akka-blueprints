# PLAN — agent-benchmark-harness

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Runner[RunnerAgent]:::agent
  Scorer[ScorerAgent]:::agent

  WF[BenchmarkRunWorkflow]:::wf
  Run[RunEntity]:::ese
  Registry[TaskRegistry]:::ese
  View[RunsView]:::view
  Consumer[TaskResultConsumer]:::cons
  Sched[RunScheduler]:::ta
  Sampler[AccuracySampler]:::ta
  API[BenchmarkEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|POST /api/runs| WF
  Sched -.->|every 24h| API
  WF -->|load tasks| Registry
  WF -->|submit task| Runner
  WF -->|score response| Scorer
  WF -->|emit events| Run
  Run -.->|projects| View
  Run -.->|subscribes| Consumer
  Consumer -->|update counters| Run
  API -->|query / SSE| View
  Sampler -.->|every 15min| Run
```

## Interaction sequence — J1 (two-task passing run)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as BenchmarkEndpoint
  participant WF as BenchmarkRunWorkflow
  participant Reg as TaskRegistry
  participant Rn as RunnerAgent
  participant Sc as ScorerAgent
  participant E as RunEntity
  participant V as RunsView

  U->>API: POST /api/runs {triggeredBy}
  API->>WF: start({runId, triggeredBy})
  WF->>E: emit RunStarted (PENDING → RUNNING)
  WF->>Reg: getTasks()
  Reg-->>WF: [t-001, t-002]

  WF->>Rn: SUBMIT_TASK(t-001)
  Rn-->>WF: TaskResponse{rawOutput}
  WF->>E: emit TaskResponseRecorded (t-001)
  WF->>Sc: SCORE_TASK(t-001, rawOutput, referenceAnswer)
  Sc-->>WF: ScoredResult{PASS, score=88}
  WF->>E: emit TaskScoredEvent (t-001, PASS)

  WF->>Rn: SUBMIT_TASK(t-002)
  Rn-->>WF: TaskResponse{rawOutput}
  WF->>E: emit TaskResponseRecorded (t-002)
  WF->>Sc: SCORE_TASK(t-002, rawOutput, referenceAnswer)
  Sc-->>WF: ScoredResult{PASS, score=92}
  WF->>E: emit TaskScoredEvent (t-002, PASS)

  Note over WF: aggregateStep — passRate=1.0 >= 0.80
  WF->>E: emit RunSummaryEvent (PASSED, passRate=1.0)
  E-->>V: project
  V-->>U: SSE update (status=PASSED)
  API-->>U: 202 {runId}
```

## State machine — `RunEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> RUNNING: RunStarted
  RUNNING --> RUNNING: TaskResponseRecorded / TaskScoredEvent
  RUNNING --> PASSED: RunSummaryEvent passRate >= threshold
  RUNNING --> FAILED: RunSummaryEvent passRate < threshold
  RUNNING --> FAILED: RunAborted (step recovery failover)
  PASSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RunEntity ||--o{ RunStarted : emits
  RunEntity ||--o{ TaskResponseRecorded : emits
  RunEntity ||--o{ TaskScoredEvent : emits
  RunEntity ||--o{ RunSummaryEvent : emits
  RunEntity ||--o{ RunAborted : emits
  RunEntity ||--o{ AccuracySnapshotRecorded : emits
  RunsView }o--|| RunEntity : projects
  TaskRegistry ||--o{ TaskRegistered : emits
  TaskResultConsumer }o--|| RunEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RunnerAgent` | `application/RunnerAgent.java` |
| `ScorerAgent` | `application/ScorerAgent.java` |
| `BenchmarkTasks` | `application/BenchmarkTasks.java` |
| `BenchmarkRunWorkflow` | `application/BenchmarkRunWorkflow.java` |
| `RunEntity` | `application/RunEntity.java` (state in `domain/BenchmarkRun.java`, events in `domain/RunEvent.java`) |
| `TaskRegistry` | `application/TaskRegistry.java` |
| `RunsView` | `application/RunsView.java` |
| `TaskResultConsumer` | `application/TaskResultConsumer.java` |
| `RunScheduler` | `application/RunScheduler.java` |
| `AccuracySampler` | `application/AccuracySampler.java` |
| `BenchmarkEndpoint` | `api/BenchmarkEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `runStep` and `scoreStep` each carry `stepTimeout(Duration.ofSeconds(90))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — the workflow degrades to `FAILED` on irrecoverable agent failure rather than hanging, with a structured `failureReason` naming the failing task and the error type.
- **Fan-out ordering:** The workflow processes tasks sequentially (one `runStep`/`scoreStep` pair per task) to stay within a single workflow thread. Parallel fan-out is a known extension point but is out of scope for the default blueprint.
- **AccuracySampler idempotency:** the sampler keys its `recordAccuracySnapshot` calls on `runId` so a tick that fires twice for the same completed run is a no-op on the entity side.
- **TaskResultConsumer idempotency:** the consumer keys counter updates on `(runId, taskId)` so a redelivered `TaskScoredEvent` does not double-count a score.
- **CI gate semantics:** `GET /api/ci-gate` reads only completed runs (PASSED or FAILED). If no completed run exists it returns `gated=true` with a `reason: "no completed run"` field. This is a conservative default — absence of evidence is treated as a failure.
- **Threshold configuration:** `benchmark.passing-threshold` (default 0.80) is read once at workflow start and stored in `RunEntity` state, so a config change mid-run does not affect an in-flight run.
