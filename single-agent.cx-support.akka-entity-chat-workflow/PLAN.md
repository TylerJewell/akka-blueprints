# PLAN — entity-workflow-chat

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
  classDef util fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ChatEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[ChatWorkflow]:::wf
  Agent[ConversationAgent]:::agent
  Compactor[ContextCompactor]:::util
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|start| Entity
  API -->|receiveMessage| Entity
  API -->|close| Entity
  Entity -.->|MessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|resume workflow| WF
  WF -->|awaitTurnStep pause| WF
  WF -->|agentStep runSingleTask| Agent
  Agent -->|AgentReply| WF
  WF -->|recordReply| Entity
  WF -->|recordStep completeTurn| Entity
  WF -->|compactStep| Compactor
  Compactor -->|ConversationSummary| WF
  WF -->|recordCompaction| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, single turn)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ChatEndpoint
  participant E as ConversationEntity
  participant S as MessageSanitizer
  participant W as ChatWorkflow
  participant A as ConversationAgent

  U->>API: POST /api/conversations
  API->>E: start(topic, userId)
  E-->>API: { conversationId }
  API->>W: start workflow "chat-{id}"
  W->>W: pause at awaitTurnStep

  U->>API: POST /api/conversations/{id}/messages
  API->>E: receiveMessage(userMessage)
  E-.->>S: MessageReceived
  S->>S: strip PII
  S->>E: attachSanitized(messageId, sanitized)
  S->>W: resume with messageId
  W->>W: advance to agentStep
  W->>E: getConversation (build context)
  E-->>W: Conversation
  W->>A: runSingleTask(instructions + context.json attachment)
  A-->>W: AgentReply
  W->>E: recordReply(reply)
  W->>E: completeTurn(turnId)
  W->>W: check token budget → loop to awaitTurnStep
  E-.->>U: SSE event(OPEN)
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> AGENT_THINKING: MessageSanitized (workflow advances)
  AGENT_THINKING --> OPEN: AgentReplied (awaiting next turn)
  OPEN --> COMPACTING: token budget exceeded
  COMPACTING --> OPEN: HistoryCompacted
  OPEN --> CLOSED: ConversationClosed
  AGENT_THINKING --> FAILED: ConversationFailed (agent error)
  CLOSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ ConversationStarted : emits
  ConversationEntity ||--o{ MessageReceived : emits
  ConversationEntity ||--o{ MessageSanitized : emits
  ConversationEntity ||--o{ AgentReplied : emits
  ConversationEntity ||--o{ HistoryCompacted : emits
  ConversationEntity ||--o{ ConversationClosed : emits
  ConversationEntity ||--o{ ConversationFailed : emits
  ConversationView }o--|| ConversationEntity : projects
  MessageSanitizer }o--|| ConversationEntity : subscribes
  ChatWorkflow }o--|| ConversationEntity : reads-and-writes
  ConversationAgent ||--o{ AgentReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `ChatWorkflow` | `application/ChatWorkflow.java` |
| `ConversationAgent` | `application/ConversationAgent.java` (tasks in `application/ChatTasks.java`) |
| `ContextCompactor` | `application/ContextCompactor.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitTurnStep` 300 s (session idle window), `agentStep` 60 s, `recordStep` 5 s, `compactStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ChatWorkflow::error)`. The 60 s on `agentStep` accommodates LLM latency (Lesson 4).
- **Workflow pause**: `awaitTurnStep` calls `Workflow.pause()`. It is resumed by `ChatEndpoint.sendMessage` via a workflow signal or equivalent mechanism after `MessageSanitizer` has written the sanitized message to the entity.
- **One agent instance per turn**: the AutonomousAgent instance id is `"agent-" + conversationId + "-" + turnId`, which gives each turn its own isolated context. `maxIterationsPerTask(3)` caps retries.
- **Compaction trigger**: the `recordStep` estimates token count as `totalChars / 4`. When the estimate exceeds 4 000, the step transitions to `compactStep` rather than looping to `awaitTurnStep`.
- **Compactor is synchronous and deterministic**: `ContextCompactor` runs in-process inside `compactStep`. No LLM call, no external service — same turn list always produces the same summary. This is a deliberate single-agent guarantee.
- **Idempotency**: `ConversationEntity.attachSanitized` is event-version-guarded — a redelivered `MessageReceived` from the Consumer does not produce a duplicate sanitize event if the message is already sanitized.
- **No saga / no compensation**: every step is either a pure entity read, an append-only event write, or a single-task agent call. Nothing external needs rollback.
