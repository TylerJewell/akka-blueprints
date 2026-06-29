# PLAN — supplier-comms-monitor

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Poller[PoOrderPoller]:::ta
  Queue[PoFeedQueue]:::ese
  Consumer[PoFeedConsumer]:::cons
  RiskAgent[DeliveryRiskAgent]:::agent
  OutreachAgent[SupplierOutreachAgent]:::agent
  WF[PoOrderWorkflow]:::wf
  Entity[PurchaseOrderEntity]:::ese
  View[PoOrderView]:::view
  EvalRunner[AccuracyEvalRunner]:::ta
  API[PoOrderEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 20s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|register + start| Entity
  Consumer -->|start workflow| WF
  WF -->|call| RiskAgent
  WF -->|call (if AT_RISK or CRITICAL)| OutreachAgent
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|approve/escalate| Entity
  API -->|query/SSE| View
  EvalRunner -.->|every 60m| Entity
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as PoOrderPoller
  participant Q as PoFeedQueue
  participant C as PoFeedConsumer
  participant E as PurchaseOrderEntity
  participant W as PoOrderWorkflow
  participant R as DeliveryRiskAgent
  participant O as SupplierOutreachAgent
  participant U as Buyer (UI)
  participant API as PoOrderEndpoint

  P->>Q: emit PoFeedReceived
  Q->>C: PoFeedReceived
  C->>E: registerPo
  C->>W: start({poId, feedEvent})
  W->>R: assess(feedEvent)
  R-->>W: AT_RISK
  W->>E: emit RiskAssessed
  W->>O: draft(feedEvent, riskAssessment)
  O-->>W: OutreachDraft (requiresBuyerApproval=true)
  W->>E: emit OutreachDrafted (status AWAITING_BUYER_APPROVAL)
  Note over E,U: Workflow pauses
  U->>API: POST /api/orders/{id}/approve
  API->>E: emit BuyerApproved + OutreachSent
  E-->>W: resume → end
```

## State machine — `PurchaseOrderEntity`

```mermaid
stateDiagram-v2
  [*] --> OPENED
  OPENED --> RISK_ASSESSED: RiskAssessed
  RISK_ASSESSED --> ON_TRACK_CONFIRMED: tier = ON_TRACK
  RISK_ASSESSED --> OUTREACH_DRAFTED: tier = AT_RISK or CRITICAL
  OUTREACH_DRAFTED --> OUTREACH_SENT: requiresBuyerApproval = false
  OUTREACH_DRAFTED --> AWAITING_BUYER_APPROVAL: requiresBuyerApproval = true
  AWAITING_BUYER_APPROVAL --> OUTREACH_SENT: buyer Approve
  AWAITING_BUYER_APPROVAL --> PROCUREMENT_REVIEW: buyer Escalate
  OUTREACH_SENT --> CLOSED
  ON_TRACK_CONFIRMED --> CLOSED
  PROCUREMENT_REVIEW --> CLOSED
  CLOSED --> [*]
```

## Entity model

```mermaid
erDiagram
  PurchaseOrderEntity ||--o{ PoOpened : emits
  PurchaseOrderEntity ||--o{ RiskAssessed : emits
  PurchaseOrderEntity ||--o{ OutreachDrafted : emits
  PurchaseOrderEntity ||--o{ BuyerApproved : emits
  PurchaseOrderEntity ||--o{ BuyerEscalated : emits
  PurchaseOrderEntity ||--o{ OutreachSent : emits
  PurchaseOrderEntity ||--o{ ProcurementReview : emits
  PurchaseOrderEntity ||--o{ OnTrackConfirmed : emits
  PurchaseOrderEntity ||--o{ PoClosed : emits
  PurchaseOrderEntity ||--o{ AccuracyEvaluated : emits
  PoOrderView }o--|| PurchaseOrderEntity : projects
  PoFeedQueue ||--o{ PoFeedReceived : emits
  PoFeedConsumer }o--|| PoFeedQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PoOrderPoller` | `application/PoOrderPoller.java` |
| `PoFeedQueue` | `application/PoFeedQueue.java` |
| `PoFeedConsumer` | `application/PoFeedConsumer.java` |
| `DeliveryRiskAgent` | `application/DeliveryRiskAgent.java` |
| `SupplierOutreachAgent` | `application/SupplierOutreachAgent.java` |
| `PoOrderWorkflow` | `application/PoOrderWorkflow.java` |
| `PurchaseOrderEntity` | `application/PurchaseOrderEntity.java` (state in `domain/PurchaseOrder.java`, events in `domain/PoEvent.java`) |
| `PoOrderView` | `application/PoOrderView.java` |
| `AccuracyEvalRunner` | `application/AccuracyEvalRunner.java` |
| `PoOrderEndpoint` | `api/PoOrderEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: risk assessment 15 s, outreach drafting 30 s. On timeout, escalate to PROCUREMENT_REVIEW.
- **HITL gate**: `PoOrderWorkflow` pauses in AWAITING_BUYER_APPROVAL using the workflow's poll-the-entity idiom; on each poll, if `decision.isPresent()` it advances.
- **Idempotency**: every workflow uses `poId` as the workflow id so duplicate feed events fold into one workflow.
- **Eval sampling**: per tick, AccuracyEvalRunner picks up to 5 CLOSED POs with no `accuracyScore`, oldest-first.
- **Guardrail position**: the `sendSupplierEmail` tool's before-call hook fires inside `PoOrderWorkflow`'s finalise step; it reads `PurchaseOrderEntity` state to confirm send eligibility before the simulated email stub is called.
