# PLAN — akka-bridge

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

  API[RunEndpoint]:::ep
  Entity[RunEntity]:::ese
  Adapter[AgentFrameworkAdapter]:::cons
  WF[RunWorkflow]:::wf
  Agent[BridgeAgent]:::agent
  Guard[ToolCallGuardrail]:::guard
  Executor[MockToolExecutor]:::guard
  View[RunView]:::view
  App[AppEndpoint]:::ep

  API -->|accept + start workflow| Entity
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|permitToolCall| Entity
  Guard -->|blockToolCall| Entity
  Entity -.->|ToolCallPermitted| Adapter
  Adapter -->|execute tool| Executor
  Executor -->|result| Adapter
  Adapter -->|recordToolResult| Entity
  Agent -->|RunResult| WF
  WF -->|complete| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as RunEndpoint
  participant E as RunEntity
  participant W as RunWorkflow
  participant A as BridgeAgent
  participant G as ToolCallGuardrail
  participant Ad as AgentFrameworkAdapter
  participant Ex as MockToolExecutor

  U->>API: POST /api/runs
  API->>E: accept(request)
  API->>W: start(runId)
  E-->>API: { runId }
  W->>E: start()
  W->>A: runSingleTask(taskDescription)
  A->>G: before-tool-call(web_search, args)
  G->>E: requestToolCall + permitToolCall
  G-->>A: permit
  E-.->>Ad: ToolCallPermitted
  Ad->>Ex: execute(web_search, args)
  Ex-->>Ad: {"results": [...]}
  Ad->>E: recordToolResult(callId, result)
  A->>G: before-tool-call(summarize, args)
  G->>E: requestToolCall + permitToolCall
  G-->>A: permit
  E-.->>Ad: ToolCallPermitted
  Ad->>Ex: execute(summarize, args)
  Ex-->>Ad: {"summary": "..."}
  Ad->>E: recordToolResult(callId, result)
  A-->>W: RunResult
  W->>E: complete(result)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `RunEntity`

```mermaid
stateDiagram-v2
  [*] --> ACCEPTED
  ACCEPTED --> RUNNING: RunStarted
  RUNNING --> COMPLETED: RunCompleted
  RUNNING --> BLOCKED: RunBlocked (budget exhausted)
  RUNNING --> FAILED: RunFailed (agent error)
  COMPLETED --> [*]
  BLOCKED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RunEntity ||--o{ RunAccepted : emits
  RunEntity ||--o{ RunStarted : emits
  RunEntity ||--o{ ToolCallRequested : emits
  RunEntity ||--o{ ToolCallPermitted : emits
  RunEntity ||--o{ ToolCallBlocked : emits
  RunEntity ||--o{ ToolResultRecorded : emits
  RunEntity ||--o{ RunCompleted : emits
  RunEntity ||--o{ RunBlocked : emits
  RunEntity ||--o{ RunFailed : emits
  RunView }o--|| RunEntity : projects
  AgentFrameworkAdapter }o--|| RunEntity : subscribes
  RunWorkflow }o--|| RunEntity : reads-and-writes
  BridgeAgent ||--o{ RunResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RunEndpoint` | `api/RunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `RunEntity` | `application/RunEntity.java` (state in `domain/Run.java`, events in `domain/RunEvent.java`) |
| `AgentFrameworkAdapter` | `application/AgentFrameworkAdapter.java` |
| `RunWorkflow` | `application/RunWorkflow.java` |
| `BridgeAgent` | `application/BridgeAgent.java` (tasks in `application/RunTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `MockToolExecutor` | `application/MockToolExecutor.java` |
| `RunView` | `application/RunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `agentStep` 120 s, `completionStep` 10 s, `error` 10 s. Default step recovery `maxRetries(1).failoverTo(RunWorkflow::error)`. The 120 s on `agentStep` accommodates multi-turn agent execution with several tool calls (Lesson 4).
- **Idempotency**: every workflow uses `"run-" + runId` as the workflow id; `RunEndpoint` starts the workflow after the `accept` command succeeds. Re-submitting the same runId returns the existing run.
- **One agent per run**: the AutonomousAgent instance id is `"bridge-" + runId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(5)` caps iterations.
- **Guardrail-driven blocking**: when `ToolCallGuardrail` rejects a call for budget exhaustion, it emits `RunBlocked` before returning the rejection. The agent receives a tool error and the run is halted. Other rejection reasons (tool not permitted, schema mismatch) feed back as tool errors that allow the agent to adapt within its remaining iterations.
- **Adapter is event-driven**: `AgentFrameworkAdapter` subscribes to `ToolCallPermitted` events. This decouples the guardrail decision (synchronous, in-agent-loop) from the actual tool execution (async Consumer). The agent loop does not block on the Consumer — it receives the tool result via the task's tool-result callback.
- **No saga / no compensation**: tool calls are append-only records. There is nothing to roll back; a blocked or failed run retains its partial tool-call log for audit.
