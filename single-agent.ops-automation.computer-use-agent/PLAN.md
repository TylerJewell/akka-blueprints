# PLAN — computer-use-agent

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
  classDef sim fill:#101028,stroke:#818cf8,color:#818cf8;

  API[TaskEndpoint]:::ep
  Entity[TaskEntity]:::ese
  GW[ConfirmationGateway]:::cons
  WF[TaskExecutionWorkflow]:::wf
  Agent[ComputerUseAgent]:::agent
  Guard[ActionGuardrail]:::guard
  Sim[EnvironmentSimulator]:::sim
  View[TaskView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|halt| Entity
  API -->|confirm| Entity
  Entity -.->|ConfirmationPending| GW
  GW -.->|enriches| View
  WF -->|startStep markRunning| Entity
  WF -->|executeActionStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|ALLOWED / BLOCKED| Agent
  Agent -->|ToolCall| WF
  WF -->|execute| Sim
  Sim -->|ScreenshotAttachment| WF
  WF -->|recordAction| Entity
  WF -->|awaitConfirmationStep poll| Entity
  WF -->|completeStep / haltStep| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path: fill-web-form)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as TaskEndpoint
  participant E as TaskEntity
  participant W as TaskExecutionWorkflow
  participant A as ComputerUseAgent
  participant G as ActionGuardrail
  participant S as EnvironmentSimulator

  U->>API: POST /api/tasks
  API->>E: submit(request)
  API->>W: start(taskId)
  E-->>API: { taskId }
  W->>E: markRunning
  loop each action
    W->>A: runSingleTask(description + screen.png attachment)
    A->>G: before-tool-call(toolCall)
    G-->>A: ALLOWED
    A-->>W: ToolCall
    W->>S: execute(toolCall)
    S-->>W: ScreenshotAttachment
    W->>E: recordAction(ActionExecuted ALLOWED)
  end
  A-->>W: TaskOutcome(SUCCESS)
  W->>E: complete(outcome)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RUNNING: TaskStarted
  RUNNING --> AWAITING_CONFIRMATION: ConfirmationPending (high-impact action)
  AWAITING_CONFIRMATION --> RUNNING: ConfirmationReceived (approved or rejected)
  RUNNING --> COMPLETED: TaskCompleted (OutcomeStatus.SUCCESS or PARTIAL)
  RUNNING --> HALTED: TaskHalted (operator halt)
  RUNNING --> FAILED: TaskFailed (agent error or timeout)
  AWAITING_CONFIRMATION --> HALTED: TaskHalted
  COMPLETED --> [*]
  HALTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskSubmitted : emits
  TaskEntity ||--o{ TaskStarted : emits
  TaskEntity ||--o{ ActionProposed : emits
  TaskEntity ||--o{ ActionExecuted : emits
  TaskEntity ||--o{ ConfirmationPending : emits
  TaskEntity ||--o{ ConfirmationReceived : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskEntity ||--o{ TaskHalted : emits
  TaskEntity ||--o{ TaskFailed : emits
  TaskView }o--|| TaskEntity : projects
  ConfirmationGateway }o--|| TaskEntity : subscribes
  TaskExecutionWorkflow }o--|| TaskEntity : reads-and-writes
  ComputerUseAgent ||--o{ ToolCall : returns
  ComputerUseAgent ||--o{ TaskOutcome : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TaskEndpoint` | `api/TaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `ConfirmationGateway` | `application/ConfirmationGateway.java` |
| `TaskExecutionWorkflow` | `application/TaskExecutionWorkflow.java` |
| `ComputerUseAgent` | `application/ComputerUseAgent.java` (tasks in `application/TasksTasks.java`) |
| `ActionGuardrail` | `application/ActionGuardrail.java` |
| `EnvironmentSimulator` | `application/EnvironmentSimulator.java` |
| `TaskView` | `application/TaskView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `startStep` 5 s, `executeActionStep` 30 s, `awaitConfirmationStep` 300 s, `completeStep` 5 s, `haltStep` 5 s, `errorStep` 5 s. The 30 s on `executeActionStep` accommodates LLM latency including screenshot processing (Lesson 4). The 300 s on `awaitConfirmationStep` gives the human 5 minutes to respond.
- **Halt-first check**: `executeActionStep` reads `task.haltRequested` before invoking the agent. If the halt flag is set, the step transitions immediately to `haltStep` — no agent call is made after a halt request.
- **Idempotency**: every workflow uses `"task-" + taskId` as the workflow id. `TaskEntity.recordAction` is event-version-guarded — a redelivered `ActionExecuted` event for a call id already in `actionHistory` is a no-op.
- **One agent per task**: the AutonomousAgent instance id is `"agent-" + taskId`, giving each task its own conversation context and iteration budget (20 max). The guardrail consumes one iteration on rejection; if all 20 are exhausted, `executeActionStep` fails over to `errorStep`.
- **Confirmation timeout**: if the user does not respond within 300 s, `awaitConfirmationStep` times out and fails over to `errorStep`; the entity transitions to `FAILED`. The pending action does NOT execute on timeout.
- **No saga / no compensation**: the environment simulator is in-process. There is nothing external to roll back. A task that fails or halts preserves its partial `actionHistory` for audit.
