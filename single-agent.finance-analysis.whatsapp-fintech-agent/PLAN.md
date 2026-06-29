# PLAN — whatsapp-fintech-agent

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
  classDef hitl fill:#1a1a2e,stroke:#6c63ff,color:#6c63ff;

  API[MessageEndpoint]:::ep
  Entity[MessageEntity]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[MessageWorkflow]:::wf
  Agent[FintechQueryAgent]:::agent
  Guard[PaymentGuardrail]:::guard
  HITL[HitlGate]:::hitl
  View[MessageView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  Entity -.->|MessageReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|respondStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|AgentResponse| WF
  WF -->|recordResponse| Entity
  WF -->|hitlStep evaluate| HITL
  HITL -->|ApprovalRequested| Entity
  API -->|approve / reject| Entity
  Entity -->|resume| WF
  WF -->|executeStep| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J3 (large transfer, HITL approve)

```mermaid
sequenceDiagram
  autonumber
  participant Op as Operator (UI)
  participant API as MessageEndpoint
  participant E as MessageEntity
  participant S as MessageSanitizer
  participant W as MessageWorkflow
  participant A as FintechQueryAgent
  participant G as PaymentGuardrail
  participant H as HitlGate

  Op->>API: POST /api/messages
  API->>E: receive(inbound)
  E-->>API: { messageId }
  E-.->>S: MessageReceived
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(messageId)
  W->>E: poll getMessage
  E-->>W: sanitized.isPresent()
  W->>E: markResponding
  W->>A: runSingleTask(context + sanitized text)
  A->>G: before-tool-call(transfer args)
  G-->>A: accept
  A-->>W: AgentResponse (pendingTransaction $1200)
  W->>E: recordResponse(response)
  W->>H: evaluate(pendingTransaction, threshold)
  H-->>W: required=true
  W->>E: requestApproval
  E-.->>Op: SSE event(AWAITING_APPROVAL)
  Op->>API: POST /api/messages/{id}/approve
  API->>E: approve(decision)
  E-->>W: resume
  W->>E: markExecuted
  E-.->>Op: SSE event(EXECUTED)
```

## State machine — `MessageEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> RESPONDING: ResponseStarted
  RESPONDING --> RESPONSE_READY: ResponseReady
  RESPONSE_READY --> AWAITING_APPROVAL: ApprovalRequested (amount >= threshold)
  RESPONSE_READY --> EXECUTED: TransactionExecuted (amount < threshold)
  AWAITING_APPROVAL --> EXECUTED: TransactionApproved
  AWAITING_APPROVAL --> HITL_REJECTED: TransactionRejected
  RESPONDING --> FAILED: MessageFailed (agent error)
  RECEIVED --> FAILED: MessageFailed (sanitizer error)
  EXECUTED --> [*]
  HITL_REJECTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  MessageEntity ||--o{ MessageReceived : emits
  MessageEntity ||--o{ MessageSanitized : emits
  MessageEntity ||--o{ ResponseStarted : emits
  MessageEntity ||--o{ ResponseReady : emits
  MessageEntity ||--o{ ApprovalRequested : emits
  MessageEntity ||--o{ TransactionApproved : emits
  MessageEntity ||--o{ TransactionRejected : emits
  MessageEntity ||--o{ TransactionExecuted : emits
  MessageEntity ||--o{ MessageFailed : emits
  MessageView }o--|| MessageEntity : projects
  MessageSanitizer }o--|| MessageEntity : subscribes
  MessageWorkflow }o--|| MessageEntity : reads-and-writes
  FintechQueryAgent ||--o{ AgentResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MessageEndpoint` | `api/MessageEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MessageEntity` | `application/MessageEntity.java` (state in `domain/Message.java`, events in `domain/MessageEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `MessageWorkflow` | `application/MessageWorkflow.java` |
| `FintechQueryAgent` | `application/FintechQueryAgent.java` (tasks in `application/MessageTasks.java`) |
| `PaymentGuardrail` | `application/PaymentGuardrail.java` |
| `HitlGate` | `application/HitlGate.java` |
| `MessageView` | `application/MessageView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `respondStep` 60 s, `hitlStep` unbounded (workflow suspends pending operator input), `executeStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(MessageWorkflow::error)`. The 60 s on `respondStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"msg-" + messageId` as the workflow id; `MessageSanitizer` redelivery is safe because `MessageEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized message is a no-op.
- **One agent per message**: the AutonomousAgent instance id is `"agent-" + messageId`. Each message gets its own conversation context. `maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `PaymentGuardrail` rejects a tool call, the rejection flows back into the agent loop as a structured error. The loop retries within `maxIterationsPerTask`; if all 3 iterations fail, the workflow fails over to `error` and the entity transitions to `FAILED`.
- **HITL is synchronous from the workflow's perspective**: `hitlStep` emits `ApprovalRequested` and the workflow suspends. The resume is triggered by an external command (`approve` / `reject`) arriving on the entity, which signals the workflow to continue via a callback. There is no polling loop in `hitlStep`.
- **No saga / no compensation**: payments in this blueprint are simulated. A real deployer would wrap `executeStep` in an idempotent external payment API call and handle compensation there.
