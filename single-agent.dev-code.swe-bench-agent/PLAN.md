# PLAN — swe-bench-agent

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

  API[BenchmarkEndpoint]:::ep
  Entity[BenchmarkTaskEntity]:::ese
  Preparer[SnapshotPreparer]:::cons
  WF[BenchmarkWorkflow]:::wf
  Agent[PatchEngineerAgent]:::agent
  Guard[PatchGuardrail]:::guard
  Gate[TestGateRunner]:::guard
  View[BenchmarkView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|TaskSubmitted| Preparer
  Preparer -->|attachPreparedSnapshot| Entity
  Preparer -->|start workflow| WF
  WF -->|awaitSnapshotStep poll| Entity
  WF -->|patchStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|PatchResult| WF
  WF -->|recordPatch| Entity
  WF -->|testGateStep run| Gate
  Gate -->|GateReport| WF
  WF -->|recordGateReport + complete/fail| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BenchmarkEndpoint
  participant E as BenchmarkTaskEntity
  participant P as SnapshotPreparer
  participant W as BenchmarkWorkflow
  participant A as PatchEngineerAgent
  participant G as PatchGuardrail
  participant T as TestGateRunner

  U->>API: POST /api/tasks
  API->>E: submit(description)
  E-->>API: { taskId }
  E-.->>P: TaskSubmitted
  P->>P: normalize snapshot + redact secrets
  P->>E: attachPreparedSnapshot
  P->>W: start(taskId)
  W->>E: poll getTask
  E-->>W: snapshot.isPresent()
  W->>E: markPatching
  W->>A: runSingleTask(bug description + snapshot attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: PatchResult
  W->>E: recordPatch(patch)
  W->>T: run(patch, snapshot)
  T-->>W: GateReport(PASS)
  W->>E: recordGateReport + complete
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `BenchmarkTaskEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SNAPSHOT_PREPARED: SnapshotPrepared
  SNAPSHOT_PREPARED --> PATCHING: PatchingStarted
  PATCHING --> PATCH_PRODUCED: PatchProduced
  PATCH_PRODUCED --> GATE_PASSED: GateReportRecorded (PASS)
  PATCH_PRODUCED --> GATE_FAILED: GateReportRecorded (FAIL)
  GATE_FAILED --> PATCHING: retry patchStep
  GATE_PASSED --> COMPLETED: TaskCompleted
  PATCHING --> FAILED: TaskFailed (guardrail exhausted)
  SUBMITTED --> FAILED: TaskFailed (preparer error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BenchmarkTaskEntity ||--o{ TaskSubmitted : emits
  BenchmarkTaskEntity ||--o{ SnapshotPrepared : emits
  BenchmarkTaskEntity ||--o{ PatchingStarted : emits
  BenchmarkTaskEntity ||--o{ PatchProduced : emits
  BenchmarkTaskEntity ||--o{ GateReportRecorded : emits
  BenchmarkTaskEntity ||--o{ TaskCompleted : emits
  BenchmarkTaskEntity ||--o{ TaskFailed : emits
  BenchmarkView }o--|| BenchmarkTaskEntity : projects
  SnapshotPreparer }o--|| BenchmarkTaskEntity : subscribes
  BenchmarkWorkflow }o--|| BenchmarkTaskEntity : reads-and-writes
  PatchEngineerAgent ||--o{ PatchResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BenchmarkEndpoint` | `api/BenchmarkEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BenchmarkTaskEntity` | `application/BenchmarkTaskEntity.java` (state in `domain/BenchmarkTask.java`, events in `domain/BenchmarkEvent.java`) |
| `SnapshotPreparer` | `application/SnapshotPreparer.java` |
| `BenchmarkWorkflow` | `application/BenchmarkWorkflow.java` |
| `PatchEngineerAgent` | `application/PatchEngineerAgent.java` (tasks in `application/BenchmarkTasks.java`) |
| `PatchGuardrail` | `application/PatchGuardrail.java` |
| `TestGateRunner` | `application/TestGateRunner.java` |
| `BenchmarkView` | `application/BenchmarkView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSnapshotStep` 15 s, `patchStep` 120 s, `testGateStep` 30 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BenchmarkWorkflow::error)`. The 120 s on `patchStep` accommodates LLM latency on larger snapshots (Lesson 4).
- **Idempotency**: every workflow uses `"bench-" + taskId` as the workflow id; the `SnapshotPreparer` Consumer is allowed to redeliver `TaskSubmitted` events because `BenchmarkTaskEntity.attachPreparedSnapshot` is event-version-guarded — a second prepare attempt against an already-prepared task is a no-op.
- **One agent per task**: the AutonomousAgent instance id is `"engineer-" + taskId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Gate-driven retry**: when `TestGateRunner` returns `GateVerdict.FAIL`, the workflow transitions the entity to `GATE_FAILED` and loops back to `patchStep` (within the step's `maxRetries(2)`). If retries are exhausted, the entity transitions to `FAILED`.
- **Gate is synchronous and deterministic**: `TestGateRunner` runs in-process inside `testGateStep`. No LLM call, no external service — the same patch always produces the same gate report.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
