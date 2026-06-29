# PLAN — code-interpreter-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef sandbox fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[JobEndpoint]:::ep
  Entity[JobEntity]:::ese
  WF[InterpretationWorkflow]:::wf
  Agent[CodeInterpreterAgent]:::agent
  Guard[CodeGuardrail]:::guard
  Sandbox[ExecutionSandbox]:::sandbox
  View[JobView]:::view
  App[AppEndpoint]:::ep

  API -->|submit + start workflow| Entity
  API -->|start| WF
  WF -->|guardStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|approve/reject| Agent
  Agent -->|ExecutionResult| WF
  WF -->|approveCode| Entity
  WF -->|executeStep run| Sandbox
  Sandbox -->|SandboxResult| WF
  WF -->|recordResult or halt| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as JobEndpoint
  participant E as JobEntity
  participant W as InterpretationWorkflow
  participant A as CodeInterpreterAgent
  participant G as CodeGuardrail
  participant SB as ExecutionSandbox

  U->>API: POST /api/jobs
  API->>E: submit(request)
  E-->>API: { jobId }
  API->>W: start(jobId)
  W->>A: runSingleTask(prompt + data attachment)
  A->>G: before-tool-call(generated code)
  G-->>A: approve
  A-->>W: ExecutionResult (code ready)
  W->>E: approveCode(generatedCode)
  W->>SB: run(pythonSource, dataPayload, 10s budget)
  SB-->>W: SandboxResult(COMPLETED)
  W->>E: recordResult(ExecutionResult)
  E-.->>U: SSE event(RESULT_RECORDED)
```

## State machine — `JobEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> CODE_APPROVED: CodeApproved
  CODE_APPROVED --> EXECUTING: ExecutionStarted
  EXECUTING --> RESULT_RECORDED: ResultRecorded
  EXECUTING --> HALTED: JobHalted (timeout / memory)
  CODE_APPROVED --> FAILED: JobFailed (agent error)
  SUBMITTED --> FAILED: JobFailed (workflow start error)
  RESULT_RECORDED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  JobEntity ||--o{ JobSubmitted : emits
  JobEntity ||--o{ CodeApproved : emits
  JobEntity ||--o{ ExecutionStarted : emits
  JobEntity ||--o{ ResultRecorded : emits
  JobEntity ||--o{ JobHalted : emits
  JobEntity ||--o{ JobFailed : emits
  JobView }o--|| JobEntity : projects
  InterpretationWorkflow }o--|| JobEntity : reads-and-writes
  CodeInterpreterAgent ||--o{ ExecutionResult : returns
  ExecutionSandbox ||--o{ SandboxResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `JobEndpoint` | `api/JobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `JobEntity` | `application/JobEntity.java` (state in `domain/Job.java`, events in `domain/JobEvent.java`) |
| `InterpretationWorkflow` | `application/InterpretationWorkflow.java` |
| `CodeInterpreterAgent` | `application/CodeInterpreterAgent.java` (tasks in `application/InterpretationTasks.java`) |
| `CodeGuardrail` | `application/CodeGuardrail.java` |
| `ExecutionSandbox` | `application/ExecutionSandbox.java` |
| `JobView` | `application/JobView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `guardStep` 60 s, `executeStep` 15 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(InterpretationWorkflow::error)`. The 60 s on `guardStep` accommodates LLM latency plus potential guardrail-triggered retries (Lesson 4).
- **Idempotency**: every workflow uses `"job-" + jobId` as the workflow id; the `JobEndpoint` starts the workflow after the entity is created. A duplicate `POST /api/jobs` with the same payload mints a new jobId, so there is no deduplication concern at the submit layer.
- **One agent per job**: the AutonomousAgent instance id is `"interpreter-" + jobId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `CodeGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations produce code that violates the guardrail, the workflow's `guardStep` fails over to `error` and the entity transitions to `FAILED`.
- **Halt is terminal**: `ExecutionSandbox` enforces wall-clock and memory budgets inside `executeStep`. On breach it calls `JobEntity.halt(breachType)` and returns. The workflow does not retry halted executions — the halt is a safety signal, not a recoverable error.
- **Sandbox is synchronous**: `ExecutionSandbox.run(...)` runs in the same thread as `executeStep` under a `ScheduledExecutorService` watchdog. No LLM call, no external service — the same code on the same data always produces the same result. This keeps the pattern's "one agent" invariant honest.
- **No saga / no compensation**: every step is either an entity write or a single-task agent call. There is nothing external to roll back.
