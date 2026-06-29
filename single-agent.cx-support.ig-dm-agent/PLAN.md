# PLAN — ig-dm-agent

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

  API[DmEndpoint]:::ep
  Entity[DmEntity]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[DmWorkflow]:::wf
  Agent[DmReplyAgent]:::agent
  Guard[ReplyGuardrail]:::guard
  View[DmView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  Entity -.->|MessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|replyStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|DmReply| WF
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
  participant API as DmEndpoint
  participant E as DmEntity
  participant S as MessageSanitizer
  participant W as DmWorkflow
  participant A as DmReplyAgent
  participant G as ReplyGuardrail

  U->>API: POST /api/messages
  API->>E: receive(inbound)
  E-->>API: { messageId }
  E-.->>S: MessageReceived
  S->>S: strip PII
  S->>E: attachSanitized
  S->>W: start(messageId)
  W->>E: poll getMessage
  E-->>W: sanitized.isPresent()
  W->>E: markReplying
  W->>A: runSingleTask(brand profile + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: DmReply
  W->>E: recordReply(reply)
  E-.->>U: SSE event(REPLIED)
```

## State machine — `DmEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> REPLYING: ReplyStarted
  REPLYING --> REPLIED: ReplyRecorded
  REPLYING --> FAILED: MessageFailed (agent error)
  RECEIVED --> FAILED: MessageFailed (sanitizer error)
  REPLIED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  DmEntity ||--o{ MessageReceived : emits
  DmEntity ||--o{ MessageSanitized : emits
  DmEntity ||--o{ ReplyStarted : emits
  DmEntity ||--o{ ReplyRecorded : emits
  DmEntity ||--o{ MessageFailed : emits
  DmView }o--|| DmEntity : projects
  MessageSanitizer }o--|| DmEntity : subscribes
  DmWorkflow }o--|| DmEntity : reads-and-writes
  DmReplyAgent ||--o{ DmReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DmEndpoint` | `api/DmEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DmEntity` | `application/DmEntity.java` (state in `domain/DmMessage.java`, events in `domain/DmEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `DmWorkflow` | `application/DmWorkflow.java` |
| `DmReplyAgent` | `application/DmReplyAgent.java` (tasks in `application/DmTasks.java`) |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `DmView` | `application/DmView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `replyStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DmWorkflow::error)`. The 60 s on `replyStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"dm-" + messageId` as the workflow id; the `MessageSanitizer` Consumer is allowed to redeliver `MessageReceived` events because `DmEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized message is a no-op.
- **One agent per message**: the AutonomousAgent instance id is `"dm-agent-" + messageId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ReplyGuardrail` rejects a candidate reply, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `replyStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
