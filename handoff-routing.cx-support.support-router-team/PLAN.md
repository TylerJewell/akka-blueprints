# PLAN — support-multi-agent

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
  Router[RouterAgent]:::agent
  Billing[BillingSpecialist]:::autonomous
  Tech[TechnicalSpecialist]:::autonomous
  Account[AccountSpecialist]:::autonomous
  Guard[DraftGuardrail]:::agent
  WF[SupportWorkflow]:::wf
  Entity[CaseEntity]:::ese
  View[CaseView]:::view
  API[SupportEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|route| Router
  WF -->|RESOLVE task| Billing
  WF -->|RESOLVE task| Tech
  WF -->|RESOLVE task| Account
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
  participant W as SupportWorkflow
  participant R as RouterAgent
  participant B as BillingSpecialist
  participant G as DraftGuardrail

  Sim->>Q: receive(InboundCase)
  Q->>S: InboundCaseReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(caseId, sanitized)
  W->>R: route(sanitized)
  R-->>W: RoutingDecision{BILLING}
  W->>E: recordRouting(BILLING) [emits CaseRouted]
  W->>B: runSingleTask(RESOLVE, prompt)
  B-->>W: Resolution
  W->>E: recordDraft(resolution) [emits ResolutionDrafted]
  W->>G: check(sanitized, resolution)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(resolution) [emits ResolutionPublished, status RESOLVED]
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
  SANITIZED --> ROUTED_BILLING: category = BILLING
  SANITIZED --> ROUTED_TECHNICAL: category = TECHNICAL
  SANITIZED --> ROUTED_ACCOUNT: category = ACCOUNT
  SANITIZED --> ESCALATED: category = UNCLEAR
  ROUTED_BILLING --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_TECHNICAL --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_ACCOUNT --> RESOLUTION_DRAFTED: ResolutionDrafted
  RESOLUTION_DRAFTED --> RESOLVED: guardrail.allowed
  RESOLUTION_DRAFTED --> BLOCKED: guardrail.rejected
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
  CaseEntity ||--o{ CaseRouted : emits
  CaseEntity ||--o{ ResolutionDrafted : emits
  CaseEntity ||--o{ GuardrailVerdictAttached : emits
  CaseEntity ||--o{ ResolutionPublished : emits
  CaseEntity ||--o{ ResolutionBlocked : emits
  CaseEntity ||--o{ CaseEscalated : emits
  CaseView }o--|| CaseEntity : projects
  TicketQueue ||--o{ InboundCaseReceived : emits
  PiiSanitizer }o--|| TicketQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TicketSimulator` | `application/TicketSimulator.java` |
| `TicketQueue` | `application/TicketQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `RouterAgent` | `application/RouterAgent.java` |
| `BillingSpecialist` | `application/BillingSpecialist.java` |
| `TechnicalSpecialist` | `application/TechnicalSpecialist.java` |
| `AccountSpecialist` | `application/AccountSpecialist.java` |
| `DraftGuardrail` | `application/DraftGuardrail.java` |
| `SupportWorkflow` | `application/SupportWorkflow.java` |
| `CaseEntity` | `application/CaseEntity.java` (state in `domain/Case.java`, events in `domain/CaseEvent.java`) |
| `CaseView` | `application/CaseView.java` |
| `SupportEndpoint` | `api/SupportEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/SupportTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `routeStep` 20 s, `guardrailStep` 20 s, `billingStep` / `technicalStep` / `accountStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the case to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-case primitive is keyed by `caseId`: `CaseEntity` id is `caseId`; `SupportWorkflow` id is `caseId`; agent sessions for `RouterAgent` and `DraftGuardrail` use `caseId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Three-way branch.** The routing step dispatches to one of three specialists (or the escalate path). Only the matched specialist is invoked; the other two see no traffic for that case.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `Resolution`, the workflow either publishes or blocks based on the guardrail verdict. There is no rollback path — a blocked draft sits in `BLOCKED` until an operator unblocks via `POST /api/cases/{id}/unblock`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks; everything else flows through to `RESOLVED` autonomously.
- **Simulator throughput.** `TicketSimulator` drips one case every 30 s; the system can comfortably process each case end-to-end inside that window with mock or real LLMs.
