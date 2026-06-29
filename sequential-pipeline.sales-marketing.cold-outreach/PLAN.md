# PLAN — cold-outreach

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef hitl fill:#1a1a00,stroke:#FFD700,color:#FFD700;

  API[OutreachEndpoint]:::ep
  Entity[ProspectEntity]:::ese
  WF[OutreachPipelineWorkflow]:::wf
  Agent[OutreachAgent]:::agent
  Research[ResearchTools]:::tool
  Draft[DraftTools]:::tool
  Send[SendTools]:::tool
  SG[SendGuardrail]:::guard
  CG[ComplianceGuardrail]:::guard
  HITL[approvalStep · HITL]:::hitl
  View[ProspectView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|researchStep runSingleTask| Agent
  Agent -.->|before-tool-call| SG
  Agent -.->|before-agent-response| CG
  Agent -->|invokes| Research
  Agent -->|invokes| Draft
  Agent -->|invokes| Send
  SG -->|recordGuardrailRejection| Entity
  CG -->|recordComplianceCheck| Entity
  Agent -->|ProspectProfile / EmailDraft / OutreachEmail| WF
  WF -->|recordProfile/Draft/Email| Entity
  WF -->|approvalStep pause| HITL
  API -->|POST review| HITL
  HITL -->|ReviewDecided| Entity
  WF -->|sendStep| Agent
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as OutreachEndpoint
  participant E as ProspectEntity
  participant W as OutreachPipelineWorkflow
  participant A as OutreachAgent
  participant SG as SendGuardrail
  participant CG as ComplianceGuardrail
  participant T as Tools (Research/Draft/Send)
  participant R as Reviewer (UI)

  U->>API: POST /api/outreach { contactEmail, companyName }
  API->>E: create(contactEmail, companyName)
  E-->>API: { prospectId }
  API->>W: start(prospectId)
  W->>E: startResearch
  W->>A: runSingleTask(RESEARCH_PROSPECT, prospect)
  A->>SG: before-tool-call(lookupFirmographics, RESEARCH)
  SG-->>A: accept (not sendEmail)
  A->>T: lookupFirmographics + fetchIntentSignals
  T-->>A: FirmographicData / List<IntentSignal>
  A-->>W: ProspectProfile
  W->>E: recordProfile
  W->>A: runSingleTask(DRAFT_EMAIL, profile)
  A->>CG: before-agent-response(emailBody)
  CG-->>A: accept (sentinels present, region allowed)
  A->>T: personalizeLine + renderTemplate
  T-->>A: EmailDraft
  A-->>W: EmailDraft
  W->>E: recordDraft + recordComplianceCheck
  W->>E: requestReview
  Note over W,R: approvalStep — workflow pauses
  R->>API: POST /api/outreach/{id}/review { approved: true }
  API->>E: recordReviewDecision
  E-->>W: ReviewDecided(approved: true)
  W->>E: startSend
  W->>A: runSingleTask(SEND_EMAIL, draft)
  A->>SG: before-tool-call(sendEmail, SEND)
  SG-->>A: accept (status SENDING, review approved)
  A->>T: sendEmail(recipient, subject, body)
  T-->>A: SendReceipt
  A-->>W: OutreachEmail
  W->>E: recordEmail
  E-.->>U: SSE event(SENT)
```

## State machine — `ProspectEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RESEARCHING: ResearchStarted
  RESEARCHING --> RESEARCHED: ProspectResearched
  RESEARCHED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: EmailDrafted
  DRAFTED --> AWAITING_REVIEW: ReviewRequested
  AWAITING_REVIEW --> SENDING: ReviewDecided(approved)
  AWAITING_REVIEW --> REVIEW_REJECTED: ReviewDecided(rejected)
  SENDING --> SENT: EmailSent
  RESEARCHING --> FAILED: OutreachFailed
  DRAFTING --> FAILED: OutreachFailed
  SENDING --> FAILED: OutreachFailed
  SENT --> [*]
  REVIEW_REJECTED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status. Only an exhausted retry budget or a step timeout transitions to `FAILED`. The `ComplianceChecked` event is also audit-only when compliance passes; the entity status advances through `DRAFTED` regardless.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  ProspectEntity ||--o{ ProspectCreated : emits
  ProspectEntity ||--o{ ResearchStarted : emits
  ProspectEntity ||--o{ ProspectResearched : emits
  ProspectEntity ||--o{ DraftStarted : emits
  ProspectEntity ||--o{ EmailDrafted : emits
  ProspectEntity ||--o{ ComplianceChecked : emits
  ProspectEntity ||--o{ ReviewRequested : emits
  ProspectEntity ||--o{ ReviewDecided : emits
  ProspectEntity ||--o{ SendStarted : emits
  ProspectEntity ||--o{ EmailSent : emits
  ProspectEntity ||--o{ GuardrailRejected : emits
  ProspectEntity ||--o{ OutreachFailed : emits
  ProspectView }o--|| ProspectEntity : projects
  OutreachPipelineWorkflow }o--|| ProspectEntity : reads-and-writes
  OutreachAgent ||--o{ ProspectProfile : returns
  OutreachAgent ||--o{ EmailDraft : returns
  OutreachAgent ||--o{ OutreachEmail : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OutreachEndpoint` | `api/OutreachEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ProspectEntity` | `application/ProspectEntity.java` (state in `domain/ProspectRecord.java`, events in `domain/OutreachEvent.java`) |
| `OutreachPipelineWorkflow` | `application/OutreachPipelineWorkflow.java` |
| `OutreachAgent` | `application/OutreachAgent.java` (tasks in `application/OutreachTasks.java`) |
| `ResearchTools` | `application/ResearchTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `SendTools` | `application/SendTools.java` |
| `SendGuardrail` | `application/SendGuardrail.java` |
| `ComplianceGuardrail` | `application/ComplianceGuardrail.java` |
| `ProspectView` | `application/ProspectView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `researchStep` 60 s, `draftStep` 60 s, `approvalStep` 86400 s (24 h), `sendStep` 30 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(OutreachPipelineWorkflow::error)`. The 60 s on agent-calling steps accommodates LLM latency including tool round-trips (Lesson 4). The 24 h on `approvalStep` gives a human reviewer a full workday; deployers can tighten this.
- **Idempotency**: each workflow uses `"pipeline-" + prospectId` as the workflow id; restart of the same prospectId is rejected by the workflow runtime. The agent instance id is `"agent-" + prospectId` so each prospect has its own per-task conversation memory.
- **One agent per prospect**: `OutreachAgent` runs three tasks per prospect — RESEARCH, DRAFT, SEND — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives both guardrails room to reject and let the agent self-correct.
- **HITL gate is synchronous in the workflow**: `approvalStep` records `ReviewRequested` on the entity and then blocks on a workflow timer. When the reviewer POSTs to `/api/outreach/{id}/review`, the endpoint writes `ReviewDecided` to the entity, and the workflow detects the state change on next poll. If the 24 h timer expires without a decision, the step fails over to `error` and the entity transitions to `FAILED`.
- **Two guardrails, one agent**: both `SendGuardrail` and `ComplianceGuardrail` are registered on `OutreachAgent`. `SendGuardrail` fires on every tool call but only acts on `sendEmail`. `ComplianceGuardrail` fires on every agent response but only acts during the DRAFT task (phase metadata check).
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed prospect stays at the last successful event; the UI shows the partial state for the user.
