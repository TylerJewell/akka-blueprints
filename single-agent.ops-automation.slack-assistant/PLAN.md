# PLAN — slack-assistant

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

  API[MessageEndpoint]:::ep
  Entity[MessageEntity]:::ese
  Sanitizer[SecretSanitizer]:::cons
  WF[MessageWorkflow]:::wf
  Agent[ChannelAssistantAgent]:::agent
  Guard[ReplyGuardrail]:::guard
  Auditor[AuditRecorder]:::guard
  View[MessageView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  Entity -.->|MessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|replyStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|AssistantReply| WF
  WF -->|recordReply| Entity
  WF -->|auditStep record| Auditor
  Auditor -->|AuditRecord| WF
  WF -->|recordAudit| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as MessageEndpoint
  participant E as MessageEntity
  participant S as SecretSanitizer
  participant W as MessageWorkflow
  participant A as ChannelAssistantAgent
  participant G as ReplyGuardrail
  participant Ar as AuditRecorder

  U->>API: POST /api/messages
  API->>E: receive(context)
  E-->>API: { messageId }
  E-.->>S: MessageReceived
  S->>S: scrub secrets
  S->>E: attachSanitized
  S->>W: start(messageId)
  W->>E: poll getMessage
  E-->>W: sanitized.isPresent()
  W->>E: markReplying
  W->>A: runSingleTask(context + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: AssistantReply
  W->>E: recordReply(reply)
  W->>Ar: record(reply metadata)
  Ar-->>W: AuditRecord
  W->>E: recordAudit(audit)
  E-.->>U: SSE event(AUDITED)
```

## State machine — `MessageEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> REPLYING: ReplyStarted
  REPLYING --> REPLY_RECORDED: ReplyRecorded
  REPLY_RECORDED --> AUDITED: AuditCompleted
  REPLYING --> FAILED: MessageFailed (agent error)
  RECEIVED --> FAILED: MessageFailed (sanitizer error)
  AUDITED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  MessageEntity ||--o{ MessageReceived : emits
  MessageEntity ||--o{ MessageSanitized : emits
  MessageEntity ||--o{ ReplyStarted : emits
  MessageEntity ||--o{ ReplyRecorded : emits
  MessageEntity ||--o{ AuditCompleted : emits
  MessageEntity ||--o{ MessageFailed : emits
  MessageView }o--|| MessageEntity : projects
  SecretSanitizer }o--|| MessageEntity : subscribes
  MessageWorkflow }o--|| MessageEntity : reads-and-writes
  ChannelAssistantAgent ||--o{ AssistantReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MessageEndpoint` | `api/MessageEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MessageEntity` | `application/MessageEntity.java` (state in `domain/Message.java`, events in `domain/MessageEvent.java`) |
| `SecretSanitizer` | `application/SecretSanitizer.java` |
| `MessageWorkflow` | `application/MessageWorkflow.java` |
| `ChannelAssistantAgent` | `application/ChannelAssistantAgent.java` (tasks in `application/MessageTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `AuditRecorder` | `application/AuditRecorder.java` |
| `MessageView` | `application/MessageView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `replyStep` 60 s, `auditStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(MessageWorkflow::error)`. The 60 s on `replyStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"msg-" + messageId` as the workflow id; `SecretSanitizer` Consumer is allowed to redeliver `MessageReceived` events because `MessageEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized message is a no-op.
- **One agent per message**: the AutonomousAgent instance id is `"assistant-" + messageId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReplyGuardrail` rejects a candidate response, the rejection returns as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `replyStep` fails over to `error` and the entity transitions to `FAILED`.
- **Audit is synchronous and deterministic**: `AuditRecorder` runs in-process inside `auditStep`. No LLM call — the same task metadata always produces the same record.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. Nothing external to roll back.
