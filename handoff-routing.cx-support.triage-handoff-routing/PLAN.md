# PLAN — triage-handoff-routing

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

  Sim[ConversationSimulator]:::ta
  Queue[ConversationQueue]:::ese
  IC[IntentClassifier]:::cons
  Triage[TriageAgent]:::agent
  RoutingG[RoutingGuardrail]:::agent
  Account[AccountSpecialist]:::autonomous
  Product[ProductSpecialist]:::autonomous
  Returns[ReturnSpecialist]:::autonomous
  ResponseG[ResponseGuardrail]:::agent
  WF[HandoffWorkflow]:::wf
  Entity[ConversationEntity]:::ese
  View[ConversationView]:::view
  API[HandoffEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| IC
  IC -->|register + normalise| Entity
  IC -->|start workflow| WF
  WF -->|triage| Triage
  WF -->|routing check| RoutingG
  WF -->|RESOLVE task| Account
  WF -->|RESOLVE task| Product
  WF -->|RESOLVE task| Returns
  WF -->|response check| ResponseG
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|review / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (account happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as ConversationSimulator
  participant Q as ConversationQueue
  participant IC as IntentClassifier
  participant E as ConversationEntity
  participant W as HandoffWorkflow
  participant T as TriageAgent
  participant RG as RoutingGuardrail
  participant A as AccountSpecialist
  participant G as ResponseGuardrail

  Sim->>Q: receive(InboundTurn)
  Q->>IC: InboundTurnReceived
  IC->>E: registerTurn + attachNormalised
  IC->>W: start(turnId, normalised)
  W->>T: triage(normalised)
  T-->>W: TriageDecision{ACCOUNT, high}
  W->>E: recordTriage(decision) [emits TurnTriaged]
  W->>RG: check(normalised, decision)
  RG-->>W: RoutingVerdict{allowed=true}
  W->>E: recordRoutingVerdict + recordRouting(ACCOUNT) [emits TurnRouted]
  W->>A: runSingleTask(RESOLVE, prompt)
  A-->>W: Reply
  W->>E: recordDraft(reply) [emits ReplyDrafted]
  W->>G: check(normalised, reply)
  G-->>W: ResponseVerdict{allowed=true}
  W->>E: publish(reply) [emits ReplyPublished, status RESOLVED]
```

The routing guardrail step (steps 7–9) runs synchronously inside the workflow before any specialist is called. A blocked routing decision short-circuits the workflow directly to `ROUTING_BLOCKED` — the specialist path is never entered.

## State machine — `ConversationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> NORMALISED: TurnNormalised
  NORMALISED --> TRIAGED: TurnTriaged
  TRIAGED --> ROUTING_BLOCKED: RoutingBlocked
  TRIAGED --> ROUTED_ACCOUNT: intent = ACCOUNT
  TRIAGED --> ROUTED_PRODUCT: intent = PRODUCT
  TRIAGED --> ROUTED_RETURNS: intent = RETURNS
  TRIAGED --> ESCALATED: intent = UNCLEAR
  ROUTING_BLOCKED --> RESOLVED: operator review(publish=true)
  ROUTING_BLOCKED --> ESCALATED: operator review(publish=false)
  ROUTED_ACCOUNT --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_PRODUCT --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_RETURNS --> REPLY_DRAFTED: ReplyDrafted
  REPLY_DRAFTED --> RESOLVED: responseVerdict.allowed
  REPLY_DRAFTED --> RESPONSE_BLOCKED: responseVerdict.rejected
  RESPONSE_BLOCKED --> RESOLVED: operator review(publish=true)
  RESPONSE_BLOCKED --> ESCALATED: operator review(publish=false)
  RESOLVED --> [*]
  ESCALATED --> [*]
```

`ResponseVerdictAttached` events do not change `status` when the verdict is `allowed=true` and the workflow proceeds to `publishStep`. The transition to `RESOLVED` comes from `ReplyPublished`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  ConversationEntity ||--o{ TurnRegistered : emits
  ConversationEntity ||--o{ TurnNormalised : emits
  ConversationEntity ||--o{ TurnTriaged : emits
  ConversationEntity ||--o{ RoutingBlocked : emits
  ConversationEntity ||--o{ TurnRouted : emits
  ConversationEntity ||--o{ ReplyDrafted : emits
  ConversationEntity ||--o{ ResponseVerdictAttached : emits
  ConversationEntity ||--o{ ReplyPublished : emits
  ConversationEntity ||--o{ ReplyBlocked : emits
  ConversationEntity ||--o{ TurnEscalated : emits
  ConversationView }o--|| ConversationEntity : projects
  ConversationQueue ||--o{ InboundTurnReceived : emits
  IntentClassifier }o--|| ConversationQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationSimulator` | `application/ConversationSimulator.java` |
| `ConversationQueue` | `application/ConversationQueue.java` |
| `IntentClassifier` | `application/IntentClassifier.java` |
| `TriageAgent` | `application/TriageAgent.java` |
| `RoutingGuardrail` | `application/RoutingGuardrail.java` |
| `AccountSpecialist` | `application/AccountSpecialist.java` |
| `ProductSpecialist` | `application/ProductSpecialist.java` |
| `ReturnSpecialist` | `application/ReturnSpecialist.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `HandoffWorkflow` | `application/HandoffWorkflow.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ConversationView` | `application/ConversationView.java` |
| `HandoffEndpoint` | `api/HandoffEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/HandoffTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `triageStep` 20 s, `routingCheckStep` 20 s, `responseCheckStep` 20 s, `accountStep` / `productStep` / `returnStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the turn to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-turn primitive is keyed by `turnId`: `ConversationEntity` id is `turnId`; `HandoffWorkflow` id is `turnId`; agent sessions for `TriageAgent`, `RoutingGuardrail`, and `ResponseGuardrail` use `turnId`. Duplicate normalise events fold into a single workflow start (workflow start is idempotent per id).
- **Two guardrail boundaries.** The routing guardrail fires synchronously inside the workflow before the specialist is ever invoked. The response guardrail fires synchronously after the specialist returns but before `ReplyPublished`. Both are blocking — the specialist is not a cost until the routing guardrail clears; the reply is not published until the response guardrail clears.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `Reply`, the workflow either publishes or blocks based on the response guardrail verdict. Blocked turns sit in `ROUTING_BLOCKED` or `RESPONSE_BLOCKED` until an operator reviews via `POST /api/turns/{id}/review`.
- **No HITL on the happy path.** The system only waits for a human when either guardrail blocks; everything else flows through to `RESOLVED` autonomously.
- **Simulator throughput.** `ConversationSimulator` drips one turn every 30 s; the system can comfortably process each turn end-to-end inside that window with mock or real LLMs.
