# PLAN — routing-classifier-pattern

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
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Sim[MessageSimulator]:::ta
  Queue[MessageQueue]:::ese
  Ingest[MessageIngestor]:::cons
  Classifier[RoutingClassifier]:::agent
  RouteGuard[RouteGuardrail]:::agent
  General[GeneralAgent]:::autonomous
  Refund[RefundAgent]:::autonomous
  Tech[TechnicalAgent]:::autonomous
  ReplyGuard[ReplyGuardrail]:::agent
  WF[RoutingWorkflow]:::wf
  Entity[MessageEntity]:::ese
  View[MessageView]:::view
  API[RoutingEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Ingest
  Ingest -->|register + start| Entity
  Ingest -->|start workflow| WF
  WF -->|classify| Classifier
  WF -->|validate route| RouteGuard
  WF -->|REPLY task| General
  WF -->|REPLY task| Refund
  WF -->|REPLY task| Tech
  WF -->|screen draft| ReplyGuard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|unblock / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (general happy path)

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
  participant I as MessageIngestor
  participant E as MessageEntity
  participant W as RoutingWorkflow
  participant C as RoutingClassifier
  participant RG as RouteGuardrail
  participant G as GeneralAgent
  participant S as ReplyGuardrail

  Sim->>Q: receive(IncomingMessage)
  Q->>I: InboundMessageReceived
  I->>E: registerMessage
  I->>W: start(messageId, incoming)
  W->>C: classify(incoming)
  C-->>W: RouteDecision{GENERAL, high}
  W->>E: recordClassification(decision) [emits MessageClassified]
  W->>RG: validate(decision)
  RG-->>W: RouteVerdict{approved=true}
  W->>E: recordRouting(GENERAL) [emits MessageRouted]
  W->>G: runSingleTask(REPLY, prompt)
  G-->>W: Reply
  W->>E: recordDraft(reply) [emits ReplyDrafted]
  W->>S: screen(incoming, reply)
  S-->>W: ReplyVerdict{allowed=true}
  W->>E: publish(reply) [emits ReplyPublished, status PUBLISHED]
```

The `RouteGuardrail` call (steps 8–9) is the before-agent-invocation gate — it happens after classification but before `GeneralAgent` is ever called. The `ReplyGuardrail` call (steps 14–15) is the before-agent-response gate — it happens after the specialist returns but before the reply is published.

## State machine — `MessageEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> CLASSIFIED: MessageClassified
  CLASSIFIED --> ROUTE_BLOCKED: RouteVerdict.approved=false
  CLASSIFIED --> ROUTED_GENERAL: route=GENERAL
  CLASSIFIED --> ROUTED_REFUND: route=REFUND
  CLASSIFIED --> ROUTED_TECHNICAL: route=TECHNICAL
  CLASSIFIED --> ABANDONED: route=UNROUTABLE
  ROUTED_GENERAL --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_REFUND --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_TECHNICAL --> REPLY_DRAFTED: ReplyDrafted
  REPLY_DRAFTED --> PUBLISHED: replyVerdict.allowed
  REPLY_DRAFTED --> REPLY_BLOCKED: replyVerdict.rejected
  REPLY_BLOCKED --> PUBLISHED: operator unblock
  PUBLISHED --> [*]
  ROUTE_BLOCKED --> [*]
  ABANDONED --> [*]
```

`ROUTE_BLOCKED` and `ABANDONED` are both terminal with no forward transition. `REPLY_BLOCKED` allows one forward transition: operator unblock → `PUBLISHED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  MessageEntity ||--o{ MessageRegistered : emits
  MessageEntity ||--o{ MessageClassified : emits
  MessageEntity ||--o{ MessageRouteBlocked : emits
  MessageEntity ||--o{ MessageRouted : emits
  MessageEntity ||--o{ ReplyDrafted : emits
  MessageEntity ||--o{ ReplyVerdictAttached : emits
  MessageEntity ||--o{ ReplyPublished : emits
  MessageEntity ||--o{ ReplyBlocked : emits
  MessageEntity ||--o{ MessageAbandoned : emits
  MessageView }o--|| MessageEntity : projects
  MessageQueue ||--o{ InboundMessageReceived : emits
  MessageIngestor }o--|| MessageQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MessageSimulator` | `application/MessageSimulator.java` |
| `MessageQueue` | `application/MessageQueue.java` |
| `MessageIngestor` | `application/MessageIngestor.java` |
| `RoutingClassifier` | `application/RoutingClassifier.java` |
| `RouteGuardrail` | `application/RouteGuardrail.java` |
| `GeneralAgent` | `application/GeneralAgent.java` |
| `RefundAgent` | `application/RefundAgent.java` |
| `TechnicalAgent` | `application/TechnicalAgent.java` |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `RoutingWorkflow` | `application/RoutingWorkflow.java` |
| `MessageEntity` | `application/MessageEntity.java` (state in `domain/Message.java`, events in `domain/MessageEvent.java`) |
| `MessageView` | `application/MessageView.java` |
| `RoutingEndpoint` | `api/RoutingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/RoutingTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `validateRouteStep` 20 s, `screenStep` 20 s, `generalStep` / `refundStep` / `technicalStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the message to `ABANDONED` with the failure reason captured.
- **Idempotency.** Every per-message primitive is keyed by `messageId`: `MessageEntity` id is `messageId`; `RoutingWorkflow` id is `messageId`; agent sessions for `RoutingClassifier`, `RouteGuardrail`, and `ReplyGuardrail` use `messageId`. Duplicate ingest events fold into a single workflow start (workflow start is idempotent per id).
- **Two guardrail positions.** `RouteGuardrail` is invoked synchronously inside `validateRouteStep` before any specialist sees the message. `ReplyGuardrail` is invoked synchronously inside `screenStep` after the specialist returns. Both are blocking gates — a failure in either stops the message from advancing to `PUBLISHED`.
- **No saga compensation.** The handoff is a single-direction transfer of ownership. Once the specialist returns its `Reply`, the workflow either publishes or blocks based on the screen verdict. A route-blocked message has no forward path; a reply-blocked message waits for operator unblock.
- **No HITL on the happy path.** Human review is only required when `ReplyGuardrail` blocks. Route-blocked messages are terminal without human action (by design — an invalid route decision should not reach a human for override).
- **Simulator throughput.** `MessageSimulator` drips one message every 30 s; the system handles each message end-to-end inside that window with mock or real model providers.
