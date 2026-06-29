# PLAN — a2a-interactions

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

  Sim[InteractionSimulator]:::ta
  Queue[InteractionQueue]:::ese
  Verifier[IdentityVerifier]:::cons
  Delegator[DelegatorAgent]:::agent
  Receiver[ReceiverAgent]:::autonomous
  Judge[HandoffJudge]:::agent
  WF[InteractionWorkflow]:::wf
  Entity[InteractionEntity]:::ese
  View[InteractionView]:::view
  Scorer[HandoffEvalScorer]:::cons
  API[InteractionEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Verifier
  Verifier -->|register + verify| Entity
  Verifier -->|start workflow| WF
  WF -->|delegate| Delegator
  WF -->|FULFILL task| Receiver
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordHandoffScore| Entity
  API -->|query / SSE| View
  API -->|submit| Queue
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (routable happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as InteractionSimulator
  participant Q as InteractionQueue
  participant V as IdentityVerifier
  participant E as InteractionEntity
  participant W as InteractionWorkflow
  participant D as DelegatorAgent
  participant R as ReceiverAgent
  participant Sc as HandoffEvalScorer
  participant J as HandoffJudge

  Sim->>Q: receive(IncomingInteraction)
  Q->>V: InboundInteractionReceived
  V->>E: registerIncoming + attachVerified(identityConfirmed=true)
  V->>W: start(interactionId, interaction)
  W->>D: route(interaction)
  D-->>W: DelegationDecision{RECEIVER}
  W->>E: recordDelegation(decision) [emits DelegationDecided]
  E->>Sc: DelegationDecided event
  Sc->>J: score(interaction, decision)
  J-->>Sc: HandoffScore
  Sc->>E: recordHandoffScore [emits HandoffScored]
  W->>E: recordRouting(RECEIVER) [emits InteractionRouted]
  W->>R: runSingleTask(FULFILL, prompt)
  R-->>W: Fulfillment
  W->>E: recordFulfillment(fulfillment) [emits FulfillmentProduced]
  W->>E: publish(fulfillment) [emits FulfillmentPublished, status FULFILLED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `HandoffEvalScorer` is a Consumer reading the entity's event stream, independent of `InteractionWorkflow`. Both writes target the same `InteractionEntity`; the entity's commands are idempotent on `interactionId`.

## State machine — `InteractionEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> VERIFIED: identityConfirmed=true
  RECEIVED --> REJECTED: identityConfirmed=false
  VERIFIED --> DELEGATED: DelegationDecided
  DELEGATED --> ROUTED: receiverTag = RECEIVER
  DELEGATED --> UNROUTABLE: receiverTag = UNROUTABLE
  ROUTED --> FULFILLMENT_PRODUCED: FulfillmentProduced
  FULFILLMENT_PRODUCED --> FULFILLED: FulfillmentPublished
  FULFILLED --> [*]
  REJECTED --> [*]
  UNROUTABLE --> [*]
```

The `HandoffScored` event does not change `status`; it attaches the eval result. The state machine therefore treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  InteractionEntity ||--o{ InteractionRegistered : emits
  InteractionEntity ||--o{ InteractionVerified : emits
  InteractionEntity ||--o{ DelegationDecided : emits
  InteractionEntity ||--o{ InteractionRouted : emits
  InteractionEntity ||--o{ FulfillmentProduced : emits
  InteractionEntity ||--o{ FulfillmentPublished : emits
  InteractionEntity ||--o{ InteractionRejected : emits
  InteractionEntity ||--o{ InteractionUnroutable : emits
  InteractionEntity ||--o{ HandoffScored : emits
  InteractionView }o--|| InteractionEntity : projects
  InteractionQueue ||--o{ InboundInteractionReceived : emits
  IdentityVerifier }o--|| InteractionQueue : subscribes
  HandoffEvalScorer }o--|| InteractionEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `InteractionSimulator` | `application/InteractionSimulator.java` |
| `InteractionQueue` | `application/InteractionQueue.java` |
| `IdentityVerifier` | `application/IdentityVerifier.java` |
| `DelegatorAgent` | `application/DelegatorAgent.java` |
| `ReceiverAgent` | `application/ReceiverAgent.java` |
| `HandoffJudge` | `application/HandoffJudge.java` |
| `InteractionWorkflow` | `application/InteractionWorkflow.java` |
| `InteractionEntity` | `application/InteractionEntity.java` (state in `domain/Interaction.java`, events in `domain/InteractionEvent.java`) |
| `InteractionView` | `application/InteractionView.java` |
| `HandoffEvalScorer` | `application/HandoffEvalScorer.java` |
| `InteractionEndpoint` | `api/InteractionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/InteractionTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `delegateStep` 20 s, `receiverStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the interaction to `UNROUTABLE` with the failure reason captured.
- **Idempotency.** Every per-interaction primitive is keyed by `interactionId`: `InteractionEntity` id is `interactionId`; `InteractionWorkflow` id is `interactionId`; agent sessions for `DelegatorAgent` and `HandoffJudge` use `interactionId`. Duplicate verification events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `HandoffEvalScorer` (Consumer) and `InteractionWorkflow` both append events to the same `InteractionEntity`. Order is not guaranteed but does not matter: `HandoffScored` only mutates `handoffScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the receiver returns its `Fulfillment`, the workflow publishes. There is no rollback path — a failed receiver step terminates in `UNROUTABLE` via default recovery.
- **Identity check is pre-LLM.** `IdentityVerifier` (Consumer) acts before any workflow or agent is called. A rejected interaction never enters the workflow and no LLM ever processes its payload. This is the principal distinction from a guardrail inside the workflow.
- **Simulator throughput.** `InteractionSimulator` drips one interaction every 30 s; the system can process each end-to-end inside that window with mock or real LLMs.
