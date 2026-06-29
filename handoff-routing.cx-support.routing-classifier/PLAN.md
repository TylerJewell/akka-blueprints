# PLAN — routing-classifier

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

  Sim[TicketSimulator]:::ta
  Queue[TicketQueue]:::ese
  Sanit[PiiSanitizer]:::cons
  Classifier[ClassifierAgent]:::agent
  Billing[BillingHandler]:::autonomous
  Tech[TechnicalHandler]:::autonomous
  Account[AccountHandler]:::autonomous
  Guard[ResponseGuardrail]:::agent
  WF[RoutingWorkflow]:::wf
  Entity[CaseEntity]:::ese
  View[CaseView]:::view
  API[RoutingEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|classify| Classifier
  WF -->|RESPOND task| Billing
  WF -->|RESPOND task| Tech
  WF -->|RESPOND task| Account
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
  participant Sim as TicketSimulator
  participant Q as TicketQueue
  participant S as PiiSanitizer
  participant E as CaseEntity
  participant W as RoutingWorkflow
  participant C as ClassifierAgent
  participant B as BillingHandler
  participant G as ResponseGuardrail

  Sim->>Q: receive(InboundTicket)
  Q->>S: InboundTicketReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(caseId, sanitized)
  W->>C: classify(sanitized)
  C-->>W: ClassificationResult{BILLING}
  W->>E: recordClassification(result) [emits CaseClassified]
  W->>E: recordRouting(BILLING) [emits CaseRouted]
  W->>B: runSingleTask(RESPOND, prompt)
  B-->>W: HandlerReply
  W->>E: recordDraft(reply) [emits ReplyDrafted]
  W->>G: check(sanitized, reply)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(reply) [emits ReplyPublished, status RESOLVED]
```

## State machine — `CaseEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: CaseSanitized
  SANITIZED --> CLASSIFIED: CaseClassified
  CLASSIFIED --> ROUTED_BILLING: category = BILLING
  CLASSIFIED --> ROUTED_TECHNICAL: category = TECHNICAL
  CLASSIFIED --> ROUTED_ACCOUNT: category = ACCOUNT
  CLASSIFIED --> ESCALATED: category = UNCLEAR
  ROUTED_BILLING --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_TECHNICAL --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_ACCOUNT --> REPLY_DRAFTED: ReplyDrafted
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
  CaseEntity ||--o{ CaseRegistered : emits
  CaseEntity ||--o{ CaseSanitized : emits
  CaseEntity ||--o{ CaseClassified : emits
  CaseEntity ||--o{ CaseRouted : emits
  CaseEntity ||--o{ ReplyDrafted : emits
  CaseEntity ||--o{ GuardrailVerdictAttached : emits
  CaseEntity ||--o{ ReplyPublished : emits
  CaseEntity ||--o{ ReplyBlocked : emits
  CaseEntity ||--o{ CaseEscalated : emits
  CaseView }o--|| CaseEntity : projects
  TicketQueue ||--o{ InboundTicketReceived : emits
  PiiSanitizer }o--|| TicketQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TicketSimulator` | `application/TicketSimulator.java` |
| `TicketQueue` | `application/TicketQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ClassifierAgent` | `application/ClassifierAgent.java` |
| `BillingHandler` | `application/BillingHandler.java` |
| `TechnicalHandler` | `application/TechnicalHandler.java` |
| `AccountHandler` | `application/AccountHandler.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `RoutingWorkflow` | `application/RoutingWorkflow.java` |
| `CaseEntity` | `application/CaseEntity.java` (state in `domain/Case.java`, events in `domain/CaseEvent.java`) |
| `CaseView` | `application/CaseView.java` |
| `RoutingEndpoint` | `api/RoutingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/RoutingTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `guardrailStep` 20 s, `billingStep` / `technicalStep` / `accountStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the case to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-case primitive is keyed by `caseId`: `CaseEntity` id is `caseId`; `RoutingWorkflow` id is `caseId`; agent sessions for `ClassifierAgent` and `ResponseGuardrail` use `caseId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the handler returns its `HandlerReply`, the workflow either publishes or blocks based on the guardrail verdict. A blocked draft sits in `BLOCKED` until an operator unblocks via `POST /api/cases/{id}/unblock`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks; everything else flows through to `RESOLVED` autonomously.
- **Three-way branch.** Unlike a two-specialist pattern, the `ACCOUNT` category adds a third branch. Each branch is structurally identical — the workflow uses the same step template with a different `AutonomousAgent` target. Adding a fourth category is therefore a mechanical extension.
- **Simulator throughput.** `TicketSimulator` drips one ticket every 30 s; the system can comfortably process each case end-to-end inside that window with mock or real LLMs.
