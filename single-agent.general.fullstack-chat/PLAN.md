# PLAN — gemini-fullstack

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
  classDef util fill:#251503,stroke:#F97316,color:#F97316;

  API[ChatEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  WF[ConversationWorkflow]:::wf
  Agent[ChatAgent]:::agent
  Builder[ConversationContextBuilder]:::util
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|create / addUserMessage| Entity
  API -->|start workflow| WF
  WF -->|generateStep buildContext| Builder
  Builder -->|conversation.json bytes| WF
  WF -->|runSingleTask attachment| Agent
  Agent -->|AgentReply| WF
  WF -->|markGenerating| Entity
  WF -->|recordReply| Entity
  Entity -.->|projects| View
  API -->|list / get / SSE| View
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
  participant B as ConversationContextBuilder
  participant A as ChatAgent

  U->>API: POST /api/conversations
  API->>E: create(title, submittedBy)
  E-->>API: { conversationId }
  U->>API: POST /api/conversations/{id}/messages
  API->>E: addUserMessage(message)
  API->>W: start(conversationId, messageId)
  W->>E: markGenerating(messageId)
  W->>B: buildContextJson(messages, windowSize)
  B-->>W: conversation.json bytes
  W->>A: runSingleTask(instructions + attachment)
  A-->>W: AgentReply
  W->>E: recordReply(messageId, reply)
  E-.->>U: SSE event(REPLY_RECORDED)
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> ACTIVE: UserMessageAdded
  ACTIVE --> AWAITING_REPLY: ReplyGenerationStarted
  AWAITING_REPLY --> REPLY_RECORDED: AgentReplyRecorded
  REPLY_RECORDED --> ACTIVE: next UserMessageAdded
  AWAITING_REPLY --> FAILED: ConversationFailed (agent error)
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationCreated : emits
  ConversationEntity ||--o{ UserMessageAdded : emits
  ConversationEntity ||--o{ ReplyGenerationStarted : emits
  ConversationEntity ||--o{ AgentReplyRecorded : emits
  ConversationEntity ||--o{ ConversationFailed : emits
  ConversationView }o--|| ConversationEntity : projects
  ConversationWorkflow }o--|| ConversationEntity : reads-and-writes
  ChatAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `ChatAgent` | `application/ChatAgent.java` (tasks in `application/ChatTasks.java`) |
| `ConversationContextBuilder` | `application/ConversationContextBuilder.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `generateStep` 60 s, `recordStep` 10 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(ConversationWorkflow::error)`. The 60 s on `generateStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: each workflow instance id is `"turn-" + conversationId + "-" + messageId`; the `ChatEndpoint` starts exactly one workflow per (conversationId, messageId) pair. A duplicate `POST /conversations/{id}/messages` with the same `messageId` is rejected by the endpoint before starting a second workflow.
- **One agent per conversation**: the AutonomousAgent instance id is `"chat-" + conversationId`, which anchors each agent's conversation context to the entity. `maxIterationsPerTask(2)` caps any internal retry within a single turn.
- **Context window management**: `ConversationContextBuilder` caps the context at the last 20 messages (configurable via `akka.javasdk.agent.context-window-size` in `application.conf`) so prompt size stays bounded as conversations grow.
- **SSE fan-out**: the `ConversationView` row is updated on every entity event; the SSE endpoint emits the full row on each update so late-joining clients receive current state immediately without replaying the event log.
- **No saga / no compensation**: `recordStep` is a single append-only write to the entity. If it fails after a retry, the entity transitions to `FAILED` with the error reason. The agent reply is not re-generated — the user can start a new turn.
