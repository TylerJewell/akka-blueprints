# PLAN — akka-handoff-routing

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
  Filter[MessageFilter]:::cons
  Guard[RoutingGuardrail]:::agent
  Triage[TriageAgent]:::agent
  Billing[BillingSpecialist]:::autonomous
  Tech[TechnicalSpecialist]:::autonomous
  Judge[HandoffJudge]:::agent
  WF[HandoffWorkflow]:::wf
  Entity[ConversationEntity]:::ese
  View[ConversationView]:::view
  Scorer[HandoffEvalScorer]:::cons
  API[HandoffEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Filter
  Filter -->|register + filter| Entity
  Filter -->|start workflow| WF
  WF -->|check context| Guard
  WF -->|classify| Triage
  WF -->|RESOLVE task| Billing
  WF -->|RESOLVE task| Tech
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordHandoffScore| Entity
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
  participant Sim as ConversationSimulator
  participant Q as ConversationQueue
  participant F as MessageFilter
  participant E as ConversationEntity
  participant W as HandoffWorkflow
  participant G as RoutingGuardrail
  participant T as TriageAgent
  participant B as BillingSpecialist
  participant Sc as HandoffEvalScorer
  participant J as HandoffJudge

  Sim->>Q: receive(InboundTurn)
  Q->>F: InboundTurnReceived
  F->>E: registerInbound + attachFiltered
  F->>W: start(conversationId, filtered)
  W->>G: check(filtered, null)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: attachGuardrailVerdict [emits GuardrailVerdictAttached]
  W->>T: classify(filtered)
  T-->>W: RoutingDecision{BILLING}
  W->>E: recordRouting(decision) [emits RoutingDecided]
  E->>Sc: RoutingDecided event
  Sc->>J: score(filtered, decision)
  J-->>Sc: HandoffScore
  Sc->>E: recordHandoffScore [emits HandoffScored]
  W->>E: recordRouted(BILLING) [emits ConversationRouted]
  W->>B: runSingleTask(RESOLVE, prompt)
  B-->>W: Reply
  W->>E: recordDraft(reply) [emits ReplyDrafted]
  W->>E: publish(reply) [emits ReplyPublished, status RESOLVED]
```

The eval-event sequence (steps 10–13) runs concurrently with the workflow's continuation — `HandoffEvalScorer` is a Consumer reading the entity's event stream, independent of `HandoffWorkflow`. Both writes target the same `ConversationEntity`; the entity's commands are idempotent on `conversationId`.

## State machine — `ConversationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> FILTERED: ConversationFiltered
  FILTERED --> GUARDRAIL_PASSED: guardrail.allowed
  FILTERED --> BLOCKED: guardrail.rejected
  GUARDRAIL_PASSED --> ROUTED_BILLING: category = BILLING
  GUARDRAIL_PASSED --> ROUTED_TECHNICAL: category = TECHNICAL
  GUARDRAIL_PASSED --> ESCALATED: category = UNCLEAR
  ROUTED_BILLING --> REPLY_READY: ReplyDrafted
  ROUTED_TECHNICAL --> REPLY_READY: ReplyDrafted
  REPLY_READY --> RESOLVED: ReplyPublished
  BLOCKED --> RESOLVED: operator unblock
  RESOLVED --> [*]
  ESCALATED --> [*]
```

The `HandoffScored` event does not change `status`; it attaches the eval result as metadata. The state machine therefore treats it as a no-op transition (omitted for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  ConversationEntity ||--o{ ConversationRegistered : emits
  ConversationEntity ||--o{ ConversationFiltered : emits
  ConversationEntity ||--o{ GuardrailVerdictAttached : emits
  ConversationEntity ||--o{ RoutingDecided : emits
  ConversationEntity ||--o{ ConversationRouted : emits
  ConversationEntity ||--o{ ReplyDrafted : emits
  ConversationEntity ||--o{ ReplyPublished : emits
  ConversationEntity ||--o{ HandoffBlocked : emits
  ConversationEntity ||--o{ ConversationEscalated : emits
  ConversationEntity ||--o{ HandoffScored : emits
  ConversationView }o--|| ConversationEntity : projects
  ConversationQueue ||--o{ InboundTurnReceived : emits
  MessageFilter }o--|| ConversationQueue : subscribes
  HandoffEvalScorer }o--|| ConversationEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationSimulator` | `application/ConversationSimulator.java` |
| `ConversationQueue` | `application/ConversationQueue.java` |
| `MessageFilter` | `application/MessageFilter.java` |
| `RoutingGuardrail` | `application/RoutingGuardrail.java` |
| `TriageAgent` | `application/TriageAgent.java` |
| `BillingSpecialist` | `application/BillingSpecialist.java` |
| `TechnicalSpecialist` | `application/TechnicalSpecialist.java` |
| `HandoffJudge` | `application/HandoffJudge.java` |
| `HandoffWorkflow` | `application/HandoffWorkflow.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ConversationView` | `application/ConversationView.java` |
| `HandoffEvalScorer` | `application/HandoffEvalScorer.java` |
| `HandoffEndpoint` | `api/HandoffEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/HandoffTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `filterConfirmStep` 20 s, `guardrailStep` 20 s, `triageStep` 20 s, `billingStep` / `technicalStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the conversation to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-conversation primitive is keyed by `conversationId`: `ConversationEntity` id is `conversationId`; `HandoffWorkflow` id is `conversationId`; agent sessions for `RoutingGuardrail`, `TriageAgent`, `HandoffJudge` use `conversationId`. Duplicate filter events fold into a single workflow start.
- **Race between eval and workflow.** `HandoffEvalScorer` (Consumer) and `HandoffWorkflow` both append events to the same `ConversationEntity`. Order is not guaranteed but does not matter: `HandoffScored` only mutates `handoffScore`, never `status`.
- **Guardrail before triage.** Unlike the response-guardrail pattern, `RoutingGuardrail` fires *before* `TriageAgent` is called. A blocked context never reaches the classifier or any specialist — the entity transitions directly to `BLOCKED` from `FILTERED`.
- **No saga compensation.** Once the specialist returns its `Reply`, the workflow publishes directly. There is no rollback — the guardrail gate is the pre-condition, not a post-check.
- **Simulator throughput.** `ConversationSimulator` drips one turn every 30 s; the system can process each conversation end-to-end inside that window with mock or real LLMs.
