# PLAN — sandboxed-code-agent

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

  API[ExecutionEndpoint]:::ep
  Entity[ExecutionEntity]:::ese
  Router[SandboxRouter]:::cons
  Monitor[SafetyHaltMonitor]:::cons
  WF[ExecutionWorkflow]:::wf
  Agent[CodeExecutionAgent]:::agent
  Guard[CodeScreeningGuardrail]:::guard
  View[ExecutionView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|screenStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|pass/reject| Agent
  Agent -->|ExecutionResult| WF
  WF -->|markRunning| Entity
  Entity -.->|CodeApproved| Router
  Router -->|dispatch| SB[(Sandbox Backend)]
  SB -->|SandboxOutput| Router
  Router -->|recordOutcome| Entity
  Entity -.->|ExecutionStarted| Monitor
  Monitor -.->|poll resource usage| SB
  Monitor -->|halt| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ExecutionEndpoint
  participant E as ExecutionEntity
  participant WF as ExecutionWorkflow
  participant A as CodeExecutionAgent
  participant G as CodeScreeningGuardrail
  participant R as SandboxRouter
  participant SB as Sandbox Backend
  participant M as SafetyHaltMonitor

  U->>API: POST /api/executions
  API->>E: submit(request)
  E-->>API: { executionId }
  API->>WF: start(executionId)
  WF->>A: runSingleTask(taskDescription)
  A->>G: before-tool-call(execute_code)
  G-->>A: pass
  A->>SB: execute_code(pythonSource)
  SB-->>A: stdout / exit code
  A-->>WF: ExecutionResult
  WF->>E: markRunning
  E-.->>R: CodeApproved
  R->>SB: dispatch(code, config)
  E-.->>M: ExecutionStarted
  M->>SB: poll resource usage
  SB-->>R: SandboxOutput
  R->>E: recordOutcome(output)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `ExecutionEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SCREENING: CodeGenerated
  SCREENING --> APPROVED: CodeApproved
  SCREENING --> SCREENING: CodeBlocked (agent retries)
  APPROVED --> RUNNING: ExecutionStarted
  RUNNING --> COMPLETED: ExecutionCompleted
  RUNNING --> HALTED: ExecutionHalted (budget breach)
  SCREENING --> FAILED: ExecutionFailed (guardrail exhausted)
  RUNNING --> FAILED: ExecutionFailed (sandbox error)
  COMPLETED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ExecutionEntity ||--o{ CodeSubmitted : emits
  ExecutionEntity ||--o{ CodeGenerated : emits
  ExecutionEntity ||--o{ CodeApproved : emits
  ExecutionEntity ||--o{ CodeBlocked : emits
  ExecutionEntity ||--o{ ExecutionStarted : emits
  ExecutionEntity ||--o{ ExecutionCompleted : emits
  ExecutionEntity ||--o{ ExecutionHalted : emits
  ExecutionEntity ||--o{ ExecutionFailed : emits
  ExecutionView }o--|| ExecutionEntity : projects
  SandboxRouter }o--|| ExecutionEntity : subscribes
  SafetyHaltMonitor }o--|| ExecutionEntity : subscribes
  ExecutionWorkflow }o--|| ExecutionEntity : reads-and-writes
  CodeExecutionAgent ||--o{ ExecutionResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ExecutionEndpoint` | `api/ExecutionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ExecutionEntity` | `application/ExecutionEntity.java` (state in `domain/Execution.java`, events in `domain/ExecutionEvent.java`) |
| `SandboxRouter` | `application/SandboxRouter.java` |
| `SafetyHaltMonitor` | `application/SafetyHaltMonitor.java` |
| `ExecutionWorkflow` | `application/ExecutionWorkflow.java` |
| `CodeExecutionAgent` | `application/CodeExecutionAgent.java` (tasks in `application/ExecutionTasks.java`) |
| `CodeScreeningGuardrail` | `application/CodeScreeningGuardrail.java` |
| `SandboxDispatcher` (interface) | `application/sandbox/SandboxDispatcher.java` |
| `DockerSandboxDispatcher` | `application/sandbox/DockerSandboxDispatcher.java` |
| `E2BSandboxDispatcher` | `application/sandbox/E2BSandboxDispatcher.java` |
| `ModalSandboxDispatcher` | `application/sandbox/ModalSandboxDispatcher.java` |
| `PlaywrightSandboxDispatcher` | `application/sandbox/PlaywrightSandboxDispatcher.java` |
| `ExecutionView` | `application/ExecutionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `screenStep` 60 s, `runStep` max(wallClockBudgetSeconds + 10, 15) s, `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ExecutionWorkflow::error)`. The 60 s on `screenStep` accommodates LLM latency including guardrail-triggered retries (Lesson 4).
- **Idempotency**: every workflow uses `"exec-" + executionId` as the workflow id; `ExecutionEntity.submit` is event-version-guarded — a duplicate submit against an already-started execution is a no-op.
- **One agent per execution**: the AutonomousAgent instance id is `"agent-" + executionId`, giving each task its own conversation context. `.maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail-driven retry**: when `CodeScreeningGuardrail` rejects a candidate tool call, the rejection surfaces as a structured tool error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail screening, `screenStep` fails over to `error` and the entity transitions to `FAILED`.
- **SafetyHaltMonitor runs independently**: the monitor's polling loop is a `ScheduledExecutorService` task, not an agent iteration. It fires based on observed sandbox resource usage. The agent may have already completed by the time a halt fires — the entity's event-version guard ensures `ExecutionHalted` is a no-op if `ExecutionCompleted` already landed.
- **SandboxRouter is pull-push**: on `CodeApproved` it reads `ExecutionEntity.getExecution()` for the code + config, dispatches to the chosen `SandboxDispatcher`, and calls `recordOutcome`. If dispatch throws, `SandboxRouter` calls `fail(reason)` on the entity.
- **No saga / no compensation**: the sandbox is fully ephemeral (Docker container or cloud microVM). There is nothing to roll back; a FAILED execution is terminal with its partial state preserved for inspection.
