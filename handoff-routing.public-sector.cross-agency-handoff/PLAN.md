# PLAN — cross-agency-case-handoff-mesh

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

  Sim[CaseSimulator]:::ta
  Inbox[CaseInbox]:::ese
  Filter[PiiScopingFilter]:::cons
  Router[JurisdictionRouter]:::agent
  Intake[IntakeAssessor]:::autonomous
  Benefits[BenefitsReviewer]:::autonomous
  Guard[JurisdictionGuardrail]:::agent
  WF[HandoffWorkflow]:::wf
  Entity[CaseEntity]:::ese
  View[CaseView]:::view
  API[CaseEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Inbox
  Inbox -.->|subscribes| Filter
  Filter -->|register + scope| Entity
  Filter -->|start workflow| WF
  WF -->|route| Router
  WF -->|verify jurisdiction| Guard
  WF -->|ASSESS task| Intake
  WF -->|REVIEW task| Benefits
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|approve / reject handoff| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (intake-to-benefits happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as CaseSimulator
  participant I as CaseInbox
  participant F as PiiScopingFilter
  participant E as CaseEntity
  participant W as HandoffWorkflow
  participant R as JurisdictionRouter
  participant G as JurisdictionGuardrail
  participant IA as IntakeAssessor
  participant Op as Operator

  Sim->>I: receive(InboundCase)
  I->>F: InboundCaseReceived
  F->>E: registerInbound + attachScoped
  F->>W: start(caseId, scoped)
  W->>R: route(scoped)
  R-->>W: RoutingDecision{INTAKE}
  W->>E: recordRouting [emits CaseRouted]
  W->>G: verify(scoped, routing)
  G-->>W: JurisdictionVerdict{allowed=true}
  W->>E: recordJurisdictionVerdict [emits JurisdictionChecked]
  W->>IA: runSingleTask(ASSESS, prompt)
  IA-->>W: SegmentOutcome{ELIGIBLE}
  W->>E: completeSegment [emits SegmentCompleted]
  W->>E: pendHandoff [emits HandoffPending]
  Op->>E: approveHandoff(approvedBy, note)
  E-->>W: HandoffEmitted
  W->>E: closeCase [emits CaseClosed, status CLOSED]
```

The approval step (steps 14–16) suspends the workflow until the operator acts. `HandoffWorkflow.approvalStep` blocks until `CaseEntity` receives an `approveHandoff` or `rejectHandoffOp` command from the `CaseEndpoint`.

## State machine — `CaseEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SCOPED: CaseScoped
  SCOPED --> ROUTED: CaseRouted
  ROUTED --> JURISDICTION_CHECK: JurisdictionChecked
  JURISDICTION_CHECK --> IN_SEGMENT: jurisdiction.allowed
  JURISDICTION_CHECK --> JURISDICTION_BLOCKED: jurisdiction.denied
  IN_SEGMENT --> SEGMENT_COMPLETE: SegmentCompleted
  SEGMENT_COMPLETE --> HANDOFF_PENDING: HandoffPending
  HANDOFF_PENDING --> HANDOFF_EMITTED: operator approves
  HANDOFF_PENDING --> HANDOFF_REJECTED: operator rejects
  HANDOFF_EMITTED --> CLOSED: CaseClosed
  ROUTED --> UNROUTABLE: segment = UNROUTABLE
  JURISDICTION_BLOCKED --> [*]
  HANDOFF_REJECTED --> [*]
  CLOSED --> [*]
  UNROUTABLE --> [*]
```

A two-segment case (intake then benefits) cycles through `HANDOFF_EMITTED → ROUTED` for the second segment before reaching `CLOSED`. The diagram shows the per-segment loop; the terminal states are shared.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  CaseEntity ||--o{ CaseRegistered : emits
  CaseEntity ||--o{ CaseScoped : emits
  CaseEntity ||--o{ CaseRouted : emits
  CaseEntity ||--o{ JurisdictionChecked : emits
  CaseEntity ||--o{ SegmentStarted : emits
  CaseEntity ||--o{ SegmentCompleted : emits
  CaseEntity ||--o{ HandoffPending : emits
  CaseEntity ||--o{ HandoffEmitted : emits
  CaseEntity ||--o{ HandoffRejected : emits
  CaseEntity ||--o{ JurisdictionBlocked : emits
  CaseEntity ||--o{ CaseClosed : emits
  CaseEntity ||--o{ CaseUnroutable : emits
  CaseView }o--|| CaseEntity : projects
  CaseInbox ||--o{ InboundCaseReceived : emits
  PiiScopingFilter }o--|| CaseInbox : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CaseSimulator` | `application/CaseSimulator.java` |
| `CaseInbox` | `application/CaseInbox.java` |
| `PiiScopingFilter` | `application/PiiScopingFilter.java` |
| `JurisdictionRouter` | `application/JurisdictionRouter.java` |
| `IntakeAssessor` | `application/IntakeAssessor.java` |
| `BenefitsReviewer` | `application/BenefitsReviewer.java` |
| `JurisdictionGuardrail` | `application/JurisdictionGuardrail.java` |
| `HandoffWorkflow` | `application/HandoffWorkflow.java` |
| `CaseEntity` | `application/CaseEntity.java` (state in `domain/Case.java`, events in `domain/CaseEvent.java`) |
| `CaseView` | `application/CaseView.java` |
| `CaseEndpoint` | `api/CaseEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/CaseTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `routeStep` 20 s, `guardStep` 20 s, `segmentStep` 60 s, `approvalStep` 60 s, `emitStep` 20 s. On timeout, default recovery is `maxRetries(2).failoverTo(error)`, which transitions the case to `UNROUTABLE` with a timeout reason captured.
- **Idempotency.** Every per-case primitive is keyed by `caseId`: `CaseEntity` id is `caseId`; `HandoffWorkflow` id is `caseId`; agent sessions for `JurisdictionRouter` and `JurisdictionGuardrail` use `caseId`. Duplicate scoping events fold into a single workflow start.
- **HITL suspension.** `approvalStep` suspends the workflow indefinitely. The case sits in `HANDOFF_PENDING` until an operator calls `approve-handoff` or `reject-handoff`. There is no auto-timeout — cases pending approval wait until an operator acts.
- **Jurisdiction guardrail fires before the agent.** `guardStep` runs before `segmentStep` in every iteration of the loop. If the guardrail blocks, `segmentStep` is never entered for that iteration; the workflow emits `JurisdictionBlocked` and terminates.
- **Sequential chain, not a fan-out.** The mesh processes one agency segment at a time. After a handoff is approved, the next segment's route → guard → agent → approval cycle begins. The two agency agents never run in parallel for the same case.
- **Simulator throughput.** `CaseSimulator` drips one case every 30 s. Each case can spend an unbounded time in `HANDOFF_PENDING`; the simulator is unaware of pending approvals. Cases accumulate if the operator is slow; the system handles any queue depth.
