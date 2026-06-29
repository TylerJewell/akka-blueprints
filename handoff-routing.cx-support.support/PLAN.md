# PLAN — support

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

  Sim[RequestSimulator]:::ta
  Queue[RequestQueue]:::ese
  Sanit[PiiSanitizer]:::cons
  Triage[TriageAgent]:::agent
  Billing[BillingSpecialist]:::autonomous
  Tech[TechnicalSpecialist]:::autonomous
  Judge[TriageJudge]:::agent
  Guard[ResponseGuardrail]:::agent
  WF[SupportWorkflow]:::wf
  Entity[TicketEntity]:::ese
  View[TicketView]:::view
  Scorer[TriageEvalScorer]:::cons
  API[SupportEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|triage| Triage
  WF -->|RESOLVE task| Billing
  WF -->|RESOLVE task| Tech
  WF -->|check draft| Guard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordTriageScore| Entity
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
  participant Sim as RequestSimulator
  participant Q as RequestQueue
  participant S as PiiSanitizer
  participant E as TicketEntity
  participant W as SupportWorkflow
  participant T as TriageAgent
  participant B as BillingSpecialist
  participant G as ResponseGuardrail
  participant Sc as TriageEvalScorer
  participant J as TriageJudge

  Sim->>Q: receive(IncomingRequest)
  Q->>S: InboundRequestReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(ticketId, sanitized)
  W->>T: triage(sanitized)
  T-->>W: TriageDecision{BILLING}
  W->>E: recordTriage(decision) [emits TriageDecided]
  E->>Sc: TriageDecided event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: TriageScore
  Sc->>E: recordTriageScore [emits TriageScored]
  W->>E: recordRouting(BILLING) [emits TicketRouted]
  W->>B: runSingleTask(RESOLVE, prompt)
  B-->>W: Resolution
  W->>E: recordDraft(resolution) [emits ResolutionDrafted]
  W->>G: check(sanitized, resolution)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(resolution) [emits ResolutionPublished, status RESOLVED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `TriageEvalScorer` is a Consumer reading the entity's event stream, independent of `SupportWorkflow`. Both writes target the same `TicketEntity`; the entity's commands are idempotent on `ticketId`.

## State machine — `TicketEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: TicketSanitized
  SANITIZED --> TRIAGED: TriageDecided
  TRIAGED --> ROUTED_BILLING: category = BILLING
  TRIAGED --> ROUTED_TECHNICAL: category = TECHNICAL
  TRIAGED --> ESCALATED: category = UNCLEAR
  ROUTED_BILLING --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_TECHNICAL --> RESOLUTION_DRAFTED: ResolutionDrafted
  RESOLUTION_DRAFTED --> RESOLVED: guardrail.allowed
  RESOLUTION_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> RESOLVED: operator unblock
  RESOLVED --> [*]
  ESCALATED --> [*]
```

The `TriageScored` event does not change `status`; it attaches the eval result. The state machine therefore treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  TicketEntity ||--o{ TicketRegistered : emits
  TicketEntity ||--o{ TicketSanitized : emits
  TicketEntity ||--o{ TriageDecided : emits
  TicketEntity ||--o{ TicketRouted : emits
  TicketEntity ||--o{ ResolutionDrafted : emits
  TicketEntity ||--o{ GuardrailVerdictAttached : emits
  TicketEntity ||--o{ ResolutionPublished : emits
  TicketEntity ||--o{ ResolutionBlocked : emits
  TicketEntity ||--o{ TicketEscalated : emits
  TicketEntity ||--o{ TriageScored : emits
  TicketView }o--|| TicketEntity : projects
  RequestQueue ||--o{ InboundRequestReceived : emits
  PiiSanitizer }o--|| RequestQueue : subscribes
  TriageEvalScorer }o--|| TicketEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RequestSimulator` | `application/RequestSimulator.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `TriageAgent` | `application/TriageAgent.java` |
| `BillingSpecialist` | `application/BillingSpecialist.java` |
| `TechnicalSpecialist` | `application/TechnicalSpecialist.java` |
| `TriageJudge` | `application/TriageJudge.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `SupportWorkflow` | `application/SupportWorkflow.java` |
| `TicketEntity` | `application/TicketEntity.java` (state in `domain/Ticket.java`, events in `domain/TicketEvent.java`) |
| `TicketView` | `application/TicketView.java` |
| `TriageEvalScorer` | `application/TriageEvalScorer.java` |
| `SupportEndpoint` | `api/SupportEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/SupportTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `triageStep` 20 s, `guardrailStep` 20 s, `billingStep` / `technicalStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the ticket to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-ticket primitive is keyed by `ticketId`: `TicketEntity` id is `ticketId`; `SupportWorkflow` id is `ticketId`; agent sessions for `TriageAgent`, `TriageJudge`, and `ResponseGuardrail` use `ticketId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `TriageEvalScorer` (Consumer) and `SupportWorkflow` both append events to the same `TicketEntity`. Order is not guaranteed but does not matter: `TriageScored` only mutates `triageScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `Resolution`, the workflow either publishes or blocks based on the guardrail verdict. There is no rollback path — a blocked draft sits in `BLOCKED` until an operator unblocks via `POST /api/tickets/{id}/unblock`.
- **No HITL on the happy path.** This is the distinction from `human-in-loop-gate`. The system only waits for a human when the guardrail blocks; everything else flows through to `RESOLVED` autonomously.
- **Simulator throughput.** `RequestSimulator` drips one request every 30 s; the system can comfortably process each ticket end-to-end inside that window with mock or real LLMs.
