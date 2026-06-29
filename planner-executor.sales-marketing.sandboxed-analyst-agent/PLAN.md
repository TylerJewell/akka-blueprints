# PLAN — sandboxed-analyst-agent

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  Analyst[AnalystAgent]:::agent
  Executor[SandboxExecutorAgent]:::agent

  WF[AnalysisWorkflow]:::wf
  Job[AnalysisJobEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[UploadQueue]:::ese
  View[AnalysisJobView]:::view
  Consumer[UploadConsumer]:::cons
  Sim[DatasetSimulator]:::ta
  Stale[StaleJobMonitor]:::ta
  API[AnalysisEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|upload| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE_REPORT| Analyst
  WF -->|EXECUTE_SCRIPT| Executor
  WF -->|emit events| Job
  WF -->|poll| Ctrl
  Job -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Job
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as AnalysisEndpoint
  participant Q as UploadQueue
  participant C as UploadConsumer
  participant W as AnalysisWorkflow
  participant A as AnalystAgent
  participant E as SandboxExecutorAgent
  participant J as AnalysisJobEntity
  participant CTL as SystemControlEntity
  participant V as AnalysisJobView

  U->>API: POST /api/jobs {datasetName, csvContent}
  API->>Q: append DatasetSubmitted
  API-->>U: 202 {jobId}
  Q->>C: DatasetSubmitted
  C->>W: start({jobId, datasetName})
  W->>J: emit JobCreated (PLANNING)
  W->>A: PLAN(datasetName, csvContent)
  A-->>W: AnalysisLedger
  W->>J: emit JobPlanned, status EXECUTING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>A: DECIDE(ledgers)
    A-->>W: Continue(StepDecision)
    W->>W: guardrail.vet(script)
    W->>E: EXECUTE_SCRIPT(pythonScript, datasetPath)
    E-->>W: ExecutionResult
    W->>W: PiiScrubber.scrub(stdout)
    W->>J: emit StepRecorded (ProgressEntry)
    W->>W: SafetyEvaluator.evaluate(entry)
  end
  W->>A: COMPOSE_REPORT
  A-->>W: AnalysisReport
  W->>J: emit JobCompleted
  J-->>V: project
  V-->>U: SSE update
```

## State machine — `AnalysisJobEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: JobPlanned
  EXECUTING --> EXECUTING: StepRecorded / StepBlocked / LedgerRevised
  EXECUTING --> COMPLETED: JobCompleted
  EXECUTING --> FAILED: JobFailed
  EXECUTING --> HALTED: JobHaltedOperator / JobHaltedAutomatic
  EXECUTING --> STUCK: JobFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  AnalysisJobEntity ||--o{ JobPlanned : emits
  AnalysisJobEntity ||--o{ StepDispatched : emits
  AnalysisJobEntity ||--o{ StepBlocked : emits
  AnalysisJobEntity ||--o{ StepRecorded : emits
  AnalysisJobEntity ||--o{ LedgerRevised : emits
  AnalysisJobEntity ||--o{ JobCompleted : emits
  AnalysisJobEntity ||--o{ JobFailed : emits
  AnalysisJobEntity ||--o{ JobHaltedAutomatic : emits
  AnalysisJobEntity ||--o{ JobHaltedOperator : emits
  AnalysisJobEntity ||--o{ JobFailedTimeout : emits
  AnalysisJobView }o--|| AnalysisJobEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  UploadQueue ||--o{ DatasetSubmitted : emits
  UploadConsumer }o--|| UploadQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AnalystAgent` | `application/AnalystAgent.java` |
| `SandboxExecutorAgent` | `application/SandboxExecutorAgent.java` |
| `AnalysisWorkflow` | `application/AnalysisWorkflow.java` |
| `AnalysisJobEntity` | `application/AnalysisJobEntity.java` (state in `domain/AnalysisJob.java`, events in `domain/AnalysisJobEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `UploadQueue` | `application/UploadQueue.java` |
| `AnalysisJobView` | `application/AnalysisJobView.java` |
| `UploadConsumer` | `application/UploadConsumer.java` |
| `DatasetSimulator` | `application/DatasetSimulator.java` |
| `StaleJobMonitor` | `application/StaleJobMonitor.java` |
| `ScriptGuardrail` | `application/ScriptGuardrail.java` |
| `PiiScrubber` | `application/PiiScrubber.java` |
| `SafetyEvaluator` | `application/SafetyEvaluator.java` |
| `AnalystTasks` | `application/AnalystTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `AnalysisEndpoint` | `api/AnalysisEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `executeStep` 120 s (covers sandbox round-trip, including E2B cold-start latency), `decideStep` 45 s, `composeReportStep` 60 s. Default recovery: `maxRetries(2).failoverTo(AnalysisWorkflow::error)`.
- **Replan budget:** the analyst may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the analyst may emit `Continue` on the same `(scriptKind, script)` at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during an `executeStep` lets the in-flight sandbox call finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `AnalysisEndpoint.upload` uses `(datasetName, uploaderEmail)` over a 10 s window to dedupe `POST /api/jobs`.
- **Stuck detection:** `StaleJobMonitor` ticks every 30 s; `JobFailedTimeout` is non-fatal to other jobs. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **Sanitizer determinism:** `PiiScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, which keeps `ProgressEntry` events deterministic and replayable.
