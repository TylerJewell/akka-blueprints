# PLAN — event-driven-planner-executor

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

  Orch[OrchestratorAgent]:::agent
  Http[HttpCallerAgent]:::agent
  Queue[QueuePublisherAgent]:::agent
  Db[DbQueryAgent]:::agent
  Script[ScriptRunnerAgent]:::agent

  WF[JobWorkflow]:::wf
  Job[JobEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  RQ[RequestQueue]:::ese
  View[JobView]:::view
  Consumer[JobRequestConsumer]:::cons
  Sim[JobSimulator]:::ta
  Stuck[StuckJobMonitor]:::ta
  API[JobEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| RQ
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| RQ
  RQ -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE_REPORT| Orch
  WF -->|HTTP_CALL| Http
  WF -->|QUEUE_PUBLISH| Queue
  WF -->|DB_QUERY| Db
  WF -->|RUN_SCRIPT| Script
  WF -->|emit events| Job
  WF -->|poll| Ctrl
  Job -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Job
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as JobEndpoint
  participant RQ as RequestQueue
  participant C as JobRequestConsumer
  participant W as JobWorkflow
  participant O as OrchestratorAgent
  participant E as Executor (Http/Queue/Db/Script)
  participant JE as JobEntity
  participant CTL as SystemControlEntity
  participant V as JobView

  U->>API: POST /api/jobs {prompt}
  API->>RQ: append JobSubmitted
  API-->>U: 202 {jobId}
  RQ->>C: JobSubmitted
  C->>W: start({jobId, prompt})
  W->>JE: emit JobCreated (PLANNING)
  W->>O: PLAN(prompt)
  O-->>W: JobLedger
  W->>JE: emit JobPlanned, status EXECUTING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>O: DECIDE(ledgers)
    O-->>W: Continue(DispatchDecision)
    W->>W: guardrail.vet(decision)
    W->>E: runSingleTask(step)
    E-->>W: StepResult
    W->>W: CredentialScrubber.scrub(content)
    W->>JE: emit StepRecorded (StepEntry)
  end
  W->>O: COMPOSE_REPORT
  O-->>W: JobReport
  W->>JE: emit JobCompleted
  JE-->>V: project
  V-->>U: SSE update
```

## State machine — `JobEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: JobPlanned
  EXECUTING --> EXECUTING: StepRecorded / StepBlocked / LedgerRevised
  EXECUTING --> COMPLETED: JobCompleted
  EXECUTING --> FAILED: JobFailed
  EXECUTING --> HALTED: JobHaltedOperator
  EXECUTING --> STUCK: JobFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  JobEntity ||--o{ JobPlanned : emits
  JobEntity ||--o{ StepDispatched : emits
  JobEntity ||--o{ StepBlocked : emits
  JobEntity ||--o{ StepRecorded : emits
  JobEntity ||--o{ LedgerRevised : emits
  JobEntity ||--o{ JobCompleted : emits
  JobEntity ||--o{ JobFailed : emits
  JobEntity ||--o{ JobHaltedOperator : emits
  JobEntity ||--o{ JobFailedTimeout : emits
  JobView }o--|| JobEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RequestQueue ||--o{ JobSubmitted : emits
  JobRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OrchestratorAgent` | `application/OrchestratorAgent.java` |
| `HttpCallerAgent` | `application/HttpCallerAgent.java` |
| `QueuePublisherAgent` | `application/QueuePublisherAgent.java` |
| `DbQueryAgent` | `application/DbQueryAgent.java` |
| `ScriptRunnerAgent` | `application/ScriptRunnerAgent.java` |
| `JobWorkflow` | `application/JobWorkflow.java` |
| `JobEntity` | `application/JobEntity.java` (state in `domain/Job.java`, events in `domain/JobEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `JobView` | `application/JobView.java` |
| `JobRequestConsumer` | `application/JobRequestConsumer.java` |
| `JobSimulator` | `application/JobSimulator.java` |
| `StuckJobMonitor` | `application/StuckJobMonitor.java` |
| `StepGuardrail` | `application/StepGuardrail.java` |
| `CredentialScrubber` | `application/CredentialScrubber.java` |
| `OrchestratorTasks` | `application/OrchestratorTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `JobEndpoint` | `api/JobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any executor call, including slack for external latency), `decideStep` 45 s, `composeReportStep` 60 s. Default recovery: `maxRetries(2).failoverTo(JobWorkflow::error)`.
- **Replan budget:** the orchestrator may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the orchestrator may emit `Continue` on the same `(executor, step)` at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight step finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `JobEndpoint.submit` uses `(prompt, requestedBy)` over a 10 s window to dedupe `POST /api/jobs`.
- **Stuck detection:** `StuckJobMonitor` ticks every 30 s; `JobFailedTimeout` is non-fatal to other jobs. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **Sanitizer determinism:** `CredentialScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, which keeps `StepEntry` events deterministic and replayable.
