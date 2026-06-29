# PLAN — function-calling-agent-baseline

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef tool fill:#0d1a1a,stroke:#2dd4bf,color:#2dd4bf;

  API[AgentRunEndpoint]:::ep
  Entity[AgentRunEntity]:::ese
  WF[AgentRunWorkflow]:::wf
  Agent[FunctionCallingAgent]:::agent
  TGuard[ToolCallGuardrail]:::guard
  AGuard[AnswerGuardrail]:::guard
  Tools[InProcessToolExecutor]:::tool
  View[AgentRunView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|markRunning| Entity
  WF -->|runStep runSingleTask| Agent
  Agent -.->|before-tool-call| TGuard
  TGuard -->|allow/block| Agent
  Agent -->|tool invocation| Tools
  Tools -->|result| Agent
  Agent -.->|before-agent-response| AGuard
  AGuard -->|allow/block| Agent
  Agent -->|AgentAnswer| WF
  WF -->|recordAnswer| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as AgentRunEndpoint
  participant E as AgentRunEntity
  participant W as AgentRunWorkflow
  participant A as FunctionCallingAgent
  participant TG as ToolCallGuardrail
  participant Tools as InProcessToolExecutor
  participant AG as AnswerGuardrail

  U->>API: POST /api/runs
  API->>E: submit(request)
  E-->>API: { runId }
  API->>W: start(runId)
  W->>E: markRunning
  W->>A: runSingleTask(query + tools)
  A->>TG: before-tool-call(calculator, args)
  TG-->>A: allow
  A->>Tools: calculator(expression)
  Tools-->>A: result
  A->>AG: before-agent-response(candidate)
  AG-->>A: allow
  A-->>W: AgentAnswer
  W->>E: recordAnswer(answer)
  E-.->>U: SSE event(ANSWER_RECORDED)
```

## State machine — `AgentRunEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RUNNING: RunStarted
  RUNNING --> ANSWER_RECORDED: AnswerRecorded
  RUNNING --> FAILED: RunFailed (agent error)
  SUBMITTED --> FAILED: RunFailed (workflow error)
  ANSWER_RECORDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  AgentRunEntity ||--o{ RunSubmitted : emits
  AgentRunEntity ||--o{ RunStarted : emits
  AgentRunEntity ||--o{ AnswerRecorded : emits
  AgentRunEntity ||--o{ RunFailed : emits
  AgentRunView }o--|| AgentRunEntity : projects
  AgentRunWorkflow }o--|| AgentRunEntity : reads-and-writes
  FunctionCallingAgent ||--o{ AgentAnswer : returns
  FunctionCallingAgent }o--|| InProcessToolExecutor : calls
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AgentRunEndpoint` | `api/AgentRunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AgentRunEntity` | `application/AgentRunEntity.java` (state in `domain/AgentRun.java`, events in `domain/AgentRunEvent.java`) |
| `AgentRunWorkflow` | `application/AgentRunWorkflow.java` |
| `FunctionCallingAgent` | `application/FunctionCallingAgent.java` (tasks in `application/AgentRunTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `AnswerGuardrail` | `application/AnswerGuardrail.java` |
| `InProcessToolExecutor` | `application/InProcessToolExecutor.java` |
| `AgentRunView` | `application/AgentRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `startStep` 5 s, `runStep` 120 s, `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AgentRunWorkflow::error)`. The 120 s on `runStep` accommodates multi-iteration tool-calling loops where each LLM call and each tool call add latency (Lesson 4).
- **Idempotency**: every workflow uses `"run-" + runId` as the workflow id. `AgentRunEntity.markRunning` is event-version-guarded — a duplicate delivery from the workflow is a no-op if the entity is already in `RUNNING`.
- **One agent per run**: the AutonomousAgent instance id is `"agent-" + runId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(6)` caps guardrail-triggered retries and tool-calling rounds at 6.
- **Guardrail-driven retry**: when `ToolCallGuardrail` blocks a tool invocation, the rejection is returned as a structured error to the agent loop so the agent can propose a corrected call. When `AnswerGuardrail` blocks a final answer, the rejection causes the agent to revise. Both consume iterations toward `maxIterationsPerTask`; if all 6 iterations are exhausted, the workflow's `runStep` fails over to `error` and the entity transitions to `FAILED`.
- **Tool execution is synchronous and deterministic**: `InProcessToolExecutor` runs in-process inside the Akka function-calling mechanism. No LLM call, no network — the same input always returns the same output. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either an append-only event write or a single-task agent call. There is nothing external to roll back.
