# PLAN — codeact-sandboxed-agent

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

  API[TaskEndpoint]:::ep
  Entity[TaskEntity]:::ese
  Sanitizer[SecretSanitizer]:::cons
  WF[ExecutionWorkflow]:::wf
  Agent[CodeActAgent]:::agent
  Guard[SandboxGuardrail]:::guard
  Halt[SafetyHaltMonitor]:::guard
  Checker[AcceptanceChecker]:::guard
  View[TaskView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  Entity -.->|CodeExecuted| Sanitizer
  Sanitizer -->|attachSanitizedOutput| Entity
  WF -->|executeStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|execute_code| WF
  WF -->|inspect output| Halt
  Halt -->|HaltTriggered| Entity
  Agent -->|TaskResolution| WF
  WF -->|recordExecution| Entity
  WF -->|inspectStep check| Checker
  Checker -->|accepted| WF
  WF -->|markSolved / markFailed| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as TaskEndpoint
  participant E as TaskEntity
  participant W as ExecutionWorkflow
  participant A as CodeActAgent
  participant G as SandboxGuardrail
  participant H as SafetyHaltMonitor
  participant Sec as SecretSanitizer
  participant Ch as AcceptanceChecker

  U->>API: POST /api/tasks
  API->>E: submit(definition)
  E-->>API: { taskId }
  API->>W: start(taskId)
  W->>A: runSingleTask(description + attachment)
  A->>G: before-tool-call(execute_code, code)
  G-->>A: accept
  A-->>W: TaskResolution (SOLVED)
  W->>H: inspect(rawOutput)
  H-->>W: Optional.empty() (clean)
  W->>E: recordExecution(snippet, rawOutput)
  E-.->>Sec: CodeExecuted
  Sec->>Sec: scrub secrets
  Sec->>E: attachSanitizedOutput
  W->>Ch: accepted?(definition, sanitizedOutput)
  Ch-->>W: true
  W->>E: markSolved(resolution)
  E-.->>U: SSE event(SOLVED)
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> EXECUTING: TaskSubmitted (workflow starts)
  EXECUTING --> EXECUTING: CodeExecuted (next iteration)
  EXECUTING --> SOLVED: TaskSolved (acceptance met)
  EXECUTING --> HALTED: HaltTriggered (safety halt)
  EXECUTING --> FAILED: TaskFailed (budget or agent error)
  SOLVED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskSubmitted : emits
  TaskEntity ||--o{ CodeExecuted : emits
  TaskEntity ||--o{ OutputSanitized : emits
  TaskEntity ||--o{ HaltTriggered : emits
  TaskEntity ||--o{ TaskSolved : emits
  TaskEntity ||--o{ TaskFailed : emits
  TaskView }o--|| TaskEntity : projects
  SecretSanitizer }o--|| TaskEntity : subscribes
  ExecutionWorkflow }o--|| TaskEntity : reads-and-writes
  CodeActAgent ||--o{ TaskResolution : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TaskEndpoint` | `api/TaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `SecretSanitizer` | `application/SecretSanitizer.java` |
| `ExecutionWorkflow` | `application/ExecutionWorkflow.java` |
| `CodeActAgent` | `application/CodeActAgent.java` (tasks in `application/CodeActTasks.java`) |
| `SandboxGuardrail` | `application/SandboxGuardrail.java` |
| `SafetyHaltMonitor` | `application/SafetyHaltMonitor.java` |
| `AcceptanceChecker` | `application/AcceptanceChecker.java` |
| `TaskView` | `application/TaskView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `executeStep` 60 s, `inspectStep` 15 s, `solvedStep` 5 s, `haltedStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ExecutionWorkflow::error)`. The 60 s on `executeStep` accommodates LLM latency plus sandbox execution time (Lesson 4).
- **Idempotency**: every workflow uses `"exec-" + taskId` as the workflow id; the `SecretSanitizer` Consumer is allowed to redeliver `CodeExecuted` events because `TaskEntity.attachSanitizedOutput` is event-version-guarded — a second sanitize attempt for the same iteration number is a no-op.
- **One agent per task**: the AutonomousAgent instance id is `"codeact-" + taskId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(5)` caps guardrail-triggered rewrites at 5 attempts total.
- **Guardrail-driven rewrite**: when `SandboxGuardrail` rejects a `before-tool-call`, the rejection is returned as a structured error to the agent loop naming the forbidden pattern. The loop counts toward `maxIterationsPerTask`; if all 5 iterations fail the guardrail, the workflow's `executeStep` fails over to `error` and the entity transitions to `FAILED`.
- **Safety halt is synchronous**: `SafetyHaltMonitor` runs inside `executeStep` after each sandbox run, before the `CodeExecuted` event is written. A halt short-circuits the event write and transitions immediately to `HALTED`. This guarantees the raw output never lands in the entity log when a halt fires.
- **AcceptanceChecker is deterministic**: no LLM call. The same sanitized output against the same acceptance criterion always yields the same result. This keeps the single-agent invariant honest.
- **No saga / no compensation**: all state transitions are append-only event writes. The sandbox is in-process and stateless between iterations.
