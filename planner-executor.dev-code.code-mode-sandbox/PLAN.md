# PLAN — code-mode-sandbox

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

  Planner[PlannerAgent]:::agent
  Executor[ExecutorAgent]:::agent

  WF[ExecutionWorkflow]:::wf
  Job[ExecutionJobEntity]:::ese
  Cap[CapabilityRegistryEntity]:::ese
  Queue[JobQueue]:::ese
  View[ExecutionJobView]:::view
  Consumer[JobRequestConsumer]:::cons
  Sim[JobSimulator]:::ta
  Stale[StaleJobMonitor]:::ta
  API[JobEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|issue/revoke| Cap
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / REVISE_PLAN / COMPOSE_OUTPUT| Planner
  WF -->|WRITE_CODE| Executor
  WF -->|gate check| Cap
  WF -->|emit events| Job
  Job -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Job
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as JobEndpoint
  participant Q as JobQueue
  participant C as JobRequestConsumer
  participant W as ExecutionWorkflow
  participant P as PlannerAgent
  participant X as ExecutorAgent
  participant E as ExecutionJobEntity
  participant Cap as CapabilityRegistryEntity
  participant V as ExecutionJobView

  U->>API: POST /api/jobs {prompt}
  API->>Q: append JobSubmitted
  API-->>U: 202 {jobId}
  Q->>C: JobSubmitted
  C->>W: start({jobId, prompt})
  W->>E: emit JobCreated (QUEUED→PLANNING)
  W->>P: PLAN(prompt)
  P-->>W: ExecutionPlan
  W->>E: emit JobPlanned, status EXECUTING
  loop for each SandboxStep
    W->>W: PromptSanitizer.scrub(codeGoal)
    W->>Cap: get("default")
    Cap-->>W: CapabilityToken
    W->>W: CapabilityGuardrail.check(step, token)
    W->>X: WRITE_CODE(sanitizedGoal)
    X-->>W: CodePayload
    W->>W: SandboxRunner.run(payload)
    W->>E: emit StepRecorded (StepRecord)
  end
  W->>P: COMPOSE_OUTPUT
  P-->>W: JobOutput
  W->>E: emit JobCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ExecutionJobEntity`

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> PLANNING: JobCreated
  PLANNING --> EXECUTING: JobPlanned
  EXECUTING --> EXECUTING: StepRecorded / StepBlocked / PlanRevised
  EXECUTING --> COMPLETED: JobCompleted
  EXECUTING --> FAILED: JobFailed
  EXECUTING --> BLOCKED: JobBlocked
  EXECUTING --> STUCK: JobFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  BLOCKED --> [*]
```

## Entity model

```mermaid
erDiagram
  ExecutionJobEntity ||--o{ JobCreated : emits
  ExecutionJobEntity ||--o{ JobPlanned : emits
  ExecutionJobEntity ||--o{ StepStarted : emits
  ExecutionJobEntity ||--o{ StepBlocked : emits
  ExecutionJobEntity ||--o{ StepRecorded : emits
  ExecutionJobEntity ||--o{ PlanRevised : emits
  ExecutionJobEntity ||--o{ JobCompleted : emits
  ExecutionJobEntity ||--o{ JobFailed : emits
  ExecutionJobEntity ||--o{ JobBlocked : emits
  ExecutionJobEntity ||--o{ JobFailedTimeout : emits
  ExecutionJobView }o--|| ExecutionJobEntity : projects
  CapabilityRegistryEntity ||--o{ TokenIssued : emits
  CapabilityRegistryEntity ||--o{ TokenRevoked : emits
  JobQueue ||--o{ JobSubmitted : emits
  JobRequestConsumer }o--|| JobQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `ExecutorAgent` | `application/ExecutorAgent.java` |
| `ExecutionWorkflow` | `application/ExecutionWorkflow.java` |
| `ExecutionJobEntity` | `application/ExecutionJobEntity.java` (state in `domain/ExecutionJob.java`, events in `domain/JobEvent.java`) |
| `CapabilityRegistryEntity` | `application/CapabilityRegistryEntity.java` |
| `JobQueue` | `application/JobQueue.java` |
| `ExecutionJobView` | `application/ExecutionJobView.java` |
| `JobRequestConsumer` | `application/JobRequestConsumer.java` |
| `JobSimulator` | `application/JobSimulator.java` |
| `StaleJobMonitor` | `application/StaleJobMonitor.java` |
| `PromptSanitizer` | `application/PromptSanitizer.java` |
| `CapabilityGuardrail` | `application/CapabilityGuardrail.java` |
| `SandboxRunner` | `application/SandboxRunner.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `JobEndpoint` | `api/JobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `revisePlanStep` 45 s, `sanitizePromptStep` 5 s, `gateStep` 10 s, `executeStep` 90 s (covers executor call + sandbox run), `decideStep` 30 s, `composeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(ExecutionWorkflow::error)`.
- **Replan budget:** the Planner may emit a revised plan at most twice without a successful step in between; a third consecutive revision is treated as a job failure.
- **Failure budget:** the executor may retry the same `(ordinal, codeGoal)` at most three times; a fourth attempt is treated as a step failure leading to job failure.
- **Gate poll:** every `gateStep` reads `CapabilityRegistryEntity.get("default")` synchronously — no caching. A token revoked while a step is in `executeStep` is caught on the next loop iteration.
- **Idempotency:** `JobEndpoint.submit` uses `(prompt, requestedBy)` over a 10 s window to deduplicate `POST /api/jobs`.
- **Stuck detection:** `StaleJobMonitor` ticks every 30 s; `JobFailedTimeout` is non-fatal to other jobs. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **Sanitizer determinism:** `PromptSanitizer.scrub` is pure and never inspects external state. The same codeGoal always yields the same sanitized string, keeping `StepRecord` events deterministic and replayable.
