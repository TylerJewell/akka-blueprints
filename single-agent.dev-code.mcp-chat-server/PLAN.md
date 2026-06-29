# PLAN — mcp-chat-server

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
  classDef mcp fill:#0e1f2a,stroke:#38bdf8,color:#38bdf8;

  API[ChatEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  WF[ChatWorkflow]:::wf
  Agent[ChatAgent]:::agent
  TGuard[ToolCallGuardrail]:::guard
  RGuard[ReplyGuardrail]:::guard
  View[ConversationView]:::view
  App[AppEndpoint]:::ep
  MCP[MockMcpServer]:::mcp

  API -->|create / receiveMessage| Entity
  API -->|start workflow| WF
  WF -->|runAgentStep runSingleTask| Agent
  Agent -.->|before-tool-call| TGuard
  TGuard -.->|allow / block| Agent
  Agent -->|MCP tool calls| MCP
  MCP -->|tool results| Agent
  Agent -.->|before-agent-response| RGuard
  RGuard -.->|pass / reject| Agent
  Agent -->|ChatReply| WF
  WF -->|recordReply / failTurn| Entity
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
  participant E as ConversationEntity
  participant W as ChatWorkflow
  participant A as ChatAgent
  participant TG as ToolCallGuardrail
  participant RG as ReplyGuardrail
  participant M as MockMcpServer

  U->>API: POST /api/conversations/{id}/messages
  API->>E: receiveMessage(message)
  E-->>API: { turnId }
  API->>W: start("turn-" + turnId)
  W->>E: markRunning(turnId)
  W->>A: runSingleTask(context + history)
  A->>TG: before-tool-call(search)
  TG-->>A: allow
  A->>M: search(query)
  M-->>A: results
  A->>RG: before-agent-response(candidate)
  RG-->>A: pass
  A-->>W: ChatReply
  W->>E: recordReply(turnId, reply)
  E-.->>U: SSE event(REPLIED)
```

## State machine — `ConversationEntity` turn lifecycle

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> RUNNING: AgentRunStarted
  RUNNING --> REPLIED: ReplyRecorded
  RUNNING --> FAILED: TurnFailed (agent error / guardrail exhaustion)
  REPLIED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationCreated : emits
  ConversationEntity ||--o{ MessageReceived : emits
  ConversationEntity ||--o{ AgentRunStarted : emits
  ConversationEntity ||--o{ ReplyRecorded : emits
  ConversationEntity ||--o{ TurnFailed : emits
  ConversationEntity ||--o{ ConversationClosed : emits
  ConversationView }o--|| ConversationEntity : projects
  ChatWorkflow }o--|| ConversationEntity : reads-and-writes
  ChatAgent ||--o{ ChatReply : returns
  ChatAgent ||--o{ ToolCall : audits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ChatWorkflow` | `application/ChatWorkflow.java` |
| `ChatAgent` | `application/ChatAgent.java` (tasks in `application/ChatTasks.java`) |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockMcpServer` | `application/MockMcpServer.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `runAgentStep` 60 s, `recordReplyStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ChatWorkflow::error)`. The 60 s on `runAgentStep` accommodates LLM latency plus MCP round-trips (Lesson 4).
- **Idempotency**: every workflow uses `"turn-" + turnId` as the workflow id; if `ChatEndpoint` receives a duplicate `POST /conversations/{id}/messages` with the same `turnId`, the second `ConversationEntity.receiveMessage` call is a no-op (event-version-guarded) and the second `start(ChatWorkflow)` is a no-op (workflow already exists).
- **One agent per conversation**: the AutonomousAgent instance id is `"agent-" + conversationId`, giving each conversation its own context window across turns. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries.
- **Blocked tool does not abort**: when `ToolCallGuardrail` blocks a call, the agent receives a structured refusal rather than an exception. The iteration counter is NOT incremented for blocked tool calls — only for rejected agent responses. This lets the agent recover gracefully within its budget.
- **Reply guardrail retry**: when `ReplyGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow's `runAgentStep` fails over to `error` and the turn transitions to `FAILED`.
- **MockMcpServer lifecycle**: the stub is started as a managed lifecycle bean before the Akka runtime begins accepting requests and shut down after the runtime drains. Graceful shutdown waits for in-flight tool calls to complete (timeout 5 s).
- **No saga / no compensation**: every step is either an entity command or a single-task agent call. There is nothing external to roll back.
