# PLAN — multi-model-chatbot

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
  Entity[ConversationEntity]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[ChatWorkflow]:::wf
  Agent[ChatAgent]:::agent
  Guard[ReplyGuardrail]:::guard
  Policy[ContentPolicyChecker]:::guard
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|receiveMessage| Entity
  Entity -.->|MessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|replyStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -->|check content policy| Policy
  Agent -->|ChatReply| WF
  WF -->|recordReply| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ChatEndpoint
  participant E as ConversationEntity
  participant S as MessageSanitizer
  participant W as ChatWorkflow
  participant A as ChatAgent
  participant G as ReplyGuardrail

  U->>API: POST /api/chat/{sessionId}/messages
  API->>E: receiveMessage(turnId, userText)
  E-->>API: { turnId }
  E-.->>S: MessageReceived
  S->>S: redact PII
  S->>E: attachSanitized(turnId, sanitizedText)
  S->>W: start(sessionId, turnId)
  W->>E: poll getSession
  E-->>W: turn.status == SANITIZED
  W->>A: runSingleTask(history + sanitizedText)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: ChatReply
  W->>E: recordReply(turnId, reply)
  E-.->>U: SSE event(REPLIED)
```

## State machine — `ConversationEntity` turn lifecycle

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> REPLIED: ReplyGenerated
  SANITIZED --> FAILED: TurnFailed (agent error)
  RECEIVED --> FAILED: TurnFailed (sanitizer error)
  REPLIED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ SessionStarted : emits
  ConversationEntity ||--o{ MessageReceived : emits
  ConversationEntity ||--o{ MessageSanitized : emits
  ConversationEntity ||--o{ ReplyGenerated : emits
  ConversationEntity ||--o{ TurnFailed : emits
  ConversationEntity ||--o{ ProviderChanged : emits
  ConversationView }o--|| ConversationEntity : projects
  MessageSanitizer }o--|| ConversationEntity : subscribes
  ChatWorkflow }o--|| ConversationEntity : reads-and-writes
  ChatAgent ||--o{ ChatReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/ConversationSession.java`, events in `domain/ConversationEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `ChatWorkflow` | `application/ChatWorkflow.java` |
| `ChatAgent` | `application/ChatAgent.java` (tasks in `application/ChatTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `ContentPolicyChecker` | `application/ContentPolicyChecker.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `replyStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ChatWorkflow::error)`. The 60 s on `replyStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"chat-" + sessionId + "-" + turnId` as the workflow id; the `MessageSanitizer` Consumer is allowed to redeliver `MessageReceived` events because `ConversationEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized turn is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"chat-" + sessionId`, which scopes the conversation context to a single session. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReplyGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `replyStep` fails over to `error` and the turn transitions to `FAILED`.
- **Content policy is synchronous and deterministic**: `ContentPolicyChecker` runs in-process inside `ReplyGuardrail`. No LLM call — same reply always produces the same check result. This is a deliberate single-agent guarantee.
- **Provider switching**: `ConversationEntity.changeProvider` emits `ProviderChanged`; subsequent `ChatWorkflow` instances read `session.activeProvider` before invoking the agent, binding the correct model configuration. Prior turn replies retain their original `providerName` in the stored `ChatReply`.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
