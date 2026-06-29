# PLAN — code-agent-chat-ui

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

  API[ChatEndpoint]:::ep
  Entity[ChatSessionEntity]:::ese
  Validator[ToolCallValidator]:::cons
  WF[ChatSessionWorkflow]:::wf
  Agent[CodeAssistantAgent]:::agent
  ToolGuard[ToolCallPolicy]:::guard
  RespGuard[ResponseGuardrail]:::guard
  View[ChatView]:::view
  App[AppEndpoint]:::ep

  API -->|create / message| Entity
  API -->|start workflow| WF
  Entity -.->|ToolCallRequested| Validator
  Validator -->|resolveToolCall| Entity
  WF -->|activeStep poll| Entity
  WF -->|runSingleTask| Agent
  Agent -.->|before-tool-call| ToolGuard
  ToolGuard -.->|approve / block| Agent
  Agent -.->|before-agent-response| RespGuard
  RespGuard -.->|pass / reject| Agent
  Agent -->|ChatResponse| WF
  WF -->|recordResponse| Entity
  Entity -.->|projects| View
  API -->|list / SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ChatEndpoint
  participant E as ChatSessionEntity
  participant WF as ChatSessionWorkflow
  participant A as CodeAssistantAgent
  participant TG as ToolCallPolicy
  participant RG as ResponseGuardrail
  participant TV as ToolCallValidator

  U->>API: POST /api/sessions/{id}/messages
  API->>E: receiveUserMessage(content)
  E-->>API: { messageId }
  E-.->>TV: ToolCallRequested (web-search)
  TV->>E: resolveToolCall(APPROVED)
  WF->>E: poll getSession
  E-->>WF: userMessage present
  WF->>A: runSingleTask(history)
  A->>TG: before-tool-call(web-search)
  TG-->>A: approve
  A->>TG: before-tool-call(code-execution)
  TG-->>A: approve
  A->>RG: before-agent-response(candidate)
  RG-->>A: pass
  A-->>WF: ChatResponse
  WF->>E: recordResponse(response)
  E-.->>U: SSE event(AssistantMessageRecorded)
```

## State machine — `ChatSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> INITIALIZING
  INITIALIZING --> ACTIVE: SessionCreated / markActive
  ACTIVE --> ACTIVE: UserMessageReceived / AssistantMessageRecorded
  ACTIVE --> IDLE: idleStep timeout
  IDLE --> ACTIVE: UserMessageReceived
  ACTIVE --> FAILED: SessionFailed (agent or guardrail error)
  IDLE --> FAILED: SessionFailed
  ACTIVE --> [*]
  IDLE --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ChatSessionEntity ||--o{ SessionCreated : emits
  ChatSessionEntity ||--o{ UserMessageReceived : emits
  ChatSessionEntity ||--o{ AssistantMessageRecorded : emits
  ChatSessionEntity ||--o{ ToolCallRequested : emits
  ChatSessionEntity ||--o{ ToolCallResolved : emits
  ChatSessionEntity ||--o{ PlanRevised : emits
  ChatSessionEntity ||--o{ SessionFailed : emits
  ChatView }o--|| ChatSessionEntity : projects
  ToolCallValidator }o--|| ChatSessionEntity : subscribes
  ChatSessionWorkflow }o--|| ChatSessionEntity : reads-and-writes
  CodeAssistantAgent ||--o{ ChatResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ChatSessionEntity` | `application/ChatSessionEntity.java` (state in `domain/ChatSession.java`, events in `domain/ChatSessionEvent.java`) |
| `ToolCallValidator` | `application/ToolCallValidator.java` |
| `ChatSessionWorkflow` | `application/ChatSessionWorkflow.java` |
| `CodeAssistantAgent` | `application/CodeAssistantAgent.java` (tasks in `application/AgentTasks.java`) |
| `ToolCallPolicy` | `application/ToolCallPolicy.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `WebSearchTool` | `application/WebSearchTool.java` |
| `CodeExecutionTool` | `application/CodeExecutionTool.java` |
| `ChatView` | `application/ChatView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `initStep` 5 s, `activeStep` 90 s, `idleStep` 30 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ChatSessionWorkflow::error)`. The 90 s on `activeStep` accommodates multi-tool LLM latency plus re-planning overhead (Lesson 4).
- **Idempotency**: `ChatSessionEntity.receiveUserMessage` is event-version-guarded; a redelivered `UserMessageReceived` for an already-active turn is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`, giving each session its own conversation context. `maxIterationsPerTask(12)` caps the planning loop at 12 iterations; guardrail rejections count toward this budget.
- **Guardrail-driven retry**: when `ToolCallPolicy` blocks a call, the rejection is returned as a structured error to the agent loop. When `ResponseGuardrail` rejects a candidate response, the agent loop retries. If all 12 iterations exhaust without a passing response, `activeStep` fails over to `error` and the entity transitions to `FAILED`.
- **Tool-call lifecycle**: every tool call is recorded via `ChatSessionEntity.ToolCallRequested` before execution and `ToolCallResolved` after. The `ToolCallValidator` Consumer handles the approval gate. This makes every tool invocation auditable regardless of the final session outcome.
- **Plan revision**: the agent emits a `PlanRevision` every 3 steps. The `ChatSessionWorkflow` records each revision via `ChatSessionEntity.recordPlanRevision`. The UI renders plan-revision badges inline in the chat thread.
- **No saga / no compensation**: each step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
