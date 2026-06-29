# PLAN — react-chatbot

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
  classDef tool fill:#1a1a2e,stroke:#6c63ff,color:#6c63ff;

  API[ChatEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  WF[ConversationWorkflow]:::wf
  Agent[ChatAgent]:::agent
  Guard[ReplyGuardrail]:::guard
  Tools[ToolRegistry]:::tool
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|startConversation| Entity
  API -->|sendMessage + start workflow| Entity
  API -->|start workflow| WF
  WF -->|agentStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -->|accept / reject| Agent
  Agent -->|tool call| Tools
  Tools -->|tool result| Agent
  Agent -->|ChatReply| WF
  WF -->|recordReply| Entity
  WF -->|failTurn on error| Entity
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
  participant W as ConversationWorkflow
  participant A as ChatAgent
  participant G as ReplyGuardrail
  participant T as ToolRegistry

  U->>API: POST /api/conversations/{id}/messages
  API->>E: sendMessage(userMessage)
  E-->>API: { turnId }
  API->>W: start(turnId)
  W->>A: runSingleTask(history + message)
  A->>T: search_knowledge_base("returns policy")
  T-->>A: [KnowledgeArticle...]
  A->>G: before-agent-response(candidate reply)
  G-->>A: accept
  A-->>W: ChatReply
  W->>E: recordReply(turnId, reply)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `ConversationEntity` turn lifecycle

```mermaid
stateDiagram-v2
  [*] --> ACTIVE : ConversationStarted
  ACTIVE --> ACTIVE : TurnStarted (new turn appended)
  ACTIVE --> ACTIVE : ReplyRecorded (turn → COMPLETED)
  ACTIVE --> ACTIVE : TurnFailed (turn → FAILED)
  ACTIVE --> CLOSED : ConversationClosed
  CLOSED --> [*]
```

## Turn state machine

```mermaid
stateDiagram-v2
  [*] --> PROCESSING : TurnStarted
  PROCESSING --> COMPLETED : ReplyRecorded
  PROCESSING --> FAILED : TurnFailed (guardrail exhaustion / agent error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationStarted : emits
  ConversationEntity ||--o{ TurnStarted : emits
  ConversationEntity ||--o{ ReplyRecorded : emits
  ConversationEntity ||--o{ TurnFailed : emits
  ConversationEntity ||--o{ ConversationClosed : emits
  ConversationView }o--|| ConversationEntity : projects
  ConversationWorkflow }o--|| ConversationEntity : reads-and-writes
  ChatAgent ||--o{ ChatReply : returns
  ToolRegistry ||--o{ ToolCall : records
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `ChatAgent` | `application/ChatAgent.java` (tasks in `application/ChatTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `ToolRegistry` | `application/ToolRegistry.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `agentStep` 60 s, `recordStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ConversationWorkflow::error)`. The 60 s on `agentStep` accommodates LLM latency plus multiple tool-call round-trips (Lesson 4).
- **One agent per turn**: the AutonomousAgent instance id is `"chat-" + conversationId + "-" + turnId`, which gives each turn its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Concurrent-turn guard**: `ChatEndpoint.sendMessage` checks `ConversationEntity.getConversation()` for an existing `PROCESSING` turn and returns `409 Conflict` before starting a new workflow. This prevents two concurrent agent calls against the same conversation state.
- **Guardrail-driven retry**: when `ReplyGuardrail` rejects a candidate reply, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, `agentStep` fails over to `error` and the entity emits `TurnFailed`.
- **Tool calls are synchronous and in-process**: `ToolRegistry` executes each tool stub from a pre-loaded JSON resource. No external service, no network call — the same tool input always returns the same stub output. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure entity read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
