# PLAN — triage-router-ui

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'fontFamily':'Instrument Sans, system-ui, sans-serif'
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef autonomous fill:#0e2a1e,stroke:#3fb950,color:#3fb950;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Sim[MessageSimulator]:::ta
  Queue[MessageQueue]:::ese
  Router[RouterAgent]:::agent
  Billing[BillingHandler]:::autonomous
  Product[ProductHandler]:::autonomous
  Guard[DraftGuardrail]:::agent
  WF[TriageWorkflow]:::wf
  Entity[MessageEntity]:::ese
  View[MessageView]:::view
  API[TriageEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|triggers| WF
  WF -->|route| Router
  WF -->|RESOLVE task| Billing
  WF -->|RESOLVE task| Product
  WF -->|check draft| Guard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|unblock / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (billing happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as MessageSimulator
  participant Q as MessageQueue
  participant W as TriageWorkflow
  participant E as MessageEntity
  participant R as RouterAgent
  participant B as BillingHandler
  participant G as DraftGuardrail

  Sim->>Q: receive(SupportMessage)
  Q->>W: start(messageId, message)
  W->>R: route(message)
  R-->>W: RouteDecision{BILLING}
  W->>E: recordRoute(decision) [emits MessageRouted{BILLING}]
  W->>B: runSingleTask(RESOLVE, prompt)
  B-->>W: HandlerReply
  W->>E: recordDraft(reply) [emits ReplyDrafted]
  W->>G: check(message, reply)
  G-->>W: GuardrailResult{allowed=true}
  W->>E: publish(reply) [emits ReplyPublished, status RESOLVED]
```

## State machine — `MessageEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> ROUTED_BILLING: category = BILLING
  RECEIVED --> ROUTED_PRODUCT: category = PRODUCT
  RECEIVED --> ESCALATED: category = UNCLEAR
  ROUTED_BILLING --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_PRODUCT --> REPLY_DRAFTED: ReplyDrafted
  REPLY_DRAFTED --> RESOLVED: guardrail.allowed
  REPLY_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> RESOLVED: operator unblock
  RESOLVED --> [*]
  ESCALATED --> [*]
```

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  MessageEntity ||--o{ MessageRegistered : emits
  MessageEntity ||--o{ MessageRouted : emits
  MessageEntity ||--o{ ReplyDrafted : emits
  MessageEntity ||--o{ GuardrailResultAttached : emits
  MessageEntity ||--o{ ReplyPublished : emits
  MessageEntity ||--o{ ReplyBlocked : emits
  MessageEntity ||--o{ MessageEscalated : emits
  MessageView }o--|| MessageEntity : projects
  MessageQueue ||--o{ InboundMessageReceived : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MessageSimulator` | `application/MessageSimulator.java` |
| `MessageQueue` | `application/MessageQueue.java` |
| `RouterAgent` | `application/RouterAgent.java` |
| `BillingHandler` | `application/BillingHandler.java` |
| `ProductHandler` | `application/ProductHandler.java` |
| `DraftGuardrail` | `application/DraftGuardrail.java` |
| `TriageWorkflow` | `application/TriageWorkflow.java` |
| `MessageEntity` | `application/MessageEntity.java` (state in `domain/Message.java`, events in `domain/MessageEvent.java`) |
| `MessageView` | `application/MessageView.java` |
| `TriageEndpoint` | `api/TriageEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/TriageTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `routeStep` 20 s, `guardrailStep` 20 s, `billingStep` / `productStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the message to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-message primitive is keyed by `messageId`: `MessageEntity` id is `messageId`; `TriageWorkflow` id is `messageId`; agent sessions for `RouterAgent` and `DraftGuardrail` use `messageId`. A duplicate simulator tick for the same `messageId` folds into a no-op on the workflow (workflow start is idempotent per id).
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the handler returns its `HandlerReply`, the workflow either publishes or blocks based on the guardrail verdict. There is no rollback path — a blocked draft sits in `BLOCKED` until an operator unblocks via `POST /api/messages/{id}/unblock`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks; everything else flows through to `RESOLVED` autonomously. Blocked messages wait indefinitely for the operator.
- **Simulator throughput.** `MessageSimulator` drips one message every 30 s; the service can process each message end-to-end inside that window with mock or real LLMs.
