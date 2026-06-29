# PLAN — ad-spend-payment-agent

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  Planner[CampaignPlannerAgent]:::agent
  Copywriter[CopywriterAgent]:::agent
  Payment[PaymentExecutorAgent]:::agent

  WF[CampaignWorkflow]:::wf
  Campaign[CampaignEntity]:::ese
  Approval[ApprovalEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[CampaignQueue]:::ese
  View[CampaignView]:::view
  Consumer[CampaignRequestConsumer]:::cons
  Sim[BriefSimulator]:::ta
  Stale[StaleCampaignMonitor]:::ta
  API[CampaignEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit brief| Queue
  API -->|halt/resume| Ctrl
  API -->|approve/reject| Approval
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE| Planner
  WF -->|WRITE_CREATIVE| Copywriter
  WF -->|EXECUTE_PAYMENT| Payment
  WF -->|emit events| Campaign
  WF -->|write/poll| Approval
  WF -->|poll| Ctrl
  Campaign -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 60s| Campaign
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as CampaignEndpoint
  participant Q as CampaignQueue
  participant C as CampaignRequestConsumer
  participant W as CampaignWorkflow
  participant P as CampaignPlannerAgent
  participant CW as CopywriterAgent
  participant PE as PaymentExecutorAgent
  participant E as CampaignEntity
  participant A as ApprovalEntity
  participant CTL as SystemControlEntity
  participant V as CampaignView

  U->>API: POST /api/campaigns {brief}
  API->>Q: append BriefSubmitted
  API-->>U: 202 {campaignId}
  Q->>C: BriefSubmitted
  C->>W: start({campaignId, brief})
  W->>E: emit CampaignCreated (PLANNING)
  W->>P: PLAN_CAMPAIGN(brief)
  P-->>W: CreativeLedger
  W->>E: emit CampaignPlanned, status EXECUTING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE_NEXT(ledgers)
    P-->>W: Continue(DispatchDecision)
    W->>W: PaymentGuardrail.vet(decision)
    alt executor = COPYWRITER
      W->>CW: WRITE_CREATIVE(task)
      CW-->>W: CreativeResult
    else executor = PAYMENT
      W->>A: requestApproval(placementId, amountWei)
      W->>E: emit PaymentApprovalRequested (AWAITING_APPROVAL)
      U->>API: POST /approvals/{placementId}/approve
      API->>A: grant(approvedBy)
      A-->>W: ApprovalGranted
      W->>PE: EXECUTE_PAYMENT(instruction)
      PE-->>W: PaymentResult
    end
    W->>W: CredentialScrubber.scrub(content)
    W->>E: emit CreativeRecorded / PaymentRecorded
  end
  W->>P: COMPOSE_REPORT
  P-->>W: CampaignReport
  W->>E: emit CampaignCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `CampaignEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: CampaignPlanned
  EXECUTING --> EXECUTING: CreativeRecorded / CreativeBlocked / LedgerRevised
  EXECUTING --> AWAITING_APPROVAL: PaymentApprovalRequested
  AWAITING_APPROVAL --> EXECUTING: PaymentApproved / PaymentRejected
  EXECUTING --> COMPLETED: CampaignCompleted
  EXECUTING --> FAILED: CampaignFailed
  EXECUTING --> HALTED: CampaignHaltedOperator
  AWAITING_APPROVAL --> STUCK: CampaignFailedTimeout
  EXECUTING --> STUCK: CampaignFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  CampaignEntity ||--o{ CampaignPlanned : emits
  CampaignEntity ||--o{ CreativeDispatched : emits
  CampaignEntity ||--o{ CreativeBlocked : emits
  CampaignEntity ||--o{ CreativeRecorded : emits
  CampaignEntity ||--o{ PaymentApprovalRequested : emits
  CampaignEntity ||--o{ PaymentApproved : emits
  CampaignEntity ||--o{ PaymentRejected : emits
  CampaignEntity ||--o{ PaymentRecorded : emits
  CampaignEntity ||--o{ LedgerRevised : emits
  CampaignEntity ||--o{ CampaignCompleted : emits
  CampaignEntity ||--o{ CampaignFailed : emits
  CampaignEntity ||--o{ CampaignHaltedOperator : emits
  CampaignEntity ||--o{ CampaignFailedTimeout : emits
  CampaignView }o--|| CampaignEntity : projects
  ApprovalEntity ||--o{ ApprovalRequested : emits
  ApprovalEntity ||--o{ ApprovalGranted : emits
  ApprovalEntity ||--o{ ApprovalRejected : emits
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  CampaignQueue ||--o{ BriefSubmitted : emits
  CampaignRequestConsumer }o--|| CampaignQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CampaignPlannerAgent` | `application/CampaignPlannerAgent.java` |
| `CopywriterAgent` | `application/CopywriterAgent.java` |
| `PaymentExecutorAgent` | `application/PaymentExecutorAgent.java` |
| `CampaignWorkflow` | `application/CampaignWorkflow.java` |
| `CampaignEntity` | `application/CampaignEntity.java` (state in `domain/Campaign.java`, events in `domain/CampaignEvent.java`) |
| `ApprovalEntity` | `application/ApprovalEntity.java` (state in `domain/ApprovalRequest.java`, events in `domain/ApprovalEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `CampaignQueue` | `application/CampaignQueue.java` |
| `CampaignView` | `application/CampaignView.java` |
| `CampaignRequestConsumer` | `application/CampaignRequestConsumer.java` |
| `BriefSimulator` | `application/BriefSimulator.java` |
| `StaleCampaignMonitor` | `application/StaleCampaignMonitor.java` |
| `PaymentGuardrail` | `application/PaymentGuardrail.java` |
| `CredentialScrubber` | `application/CredentialScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `CampaignEndpoint` | `api/CampaignEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `approvalGateStep` 600 s (accommodates a human approver; the monitor will mark the campaign STUCK if no action is taken within 10 minutes), `dispatchStep` 120 s, `decideStep` 45 s, `composeReportStep` 60 s. Default recovery: `maxRetries(2).failoverTo(CampaignWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice consecutively; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same `(executor, task)` pair at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** `checkHaltStep` reads `SystemControlEntity.get` synchronously before every payment dispatch — no caching. An operator halt arriving while a copywriter task is in-flight lets the copywriter finish; the loop exits at the next `checkHaltStep`.
- **Approval gate idempotency:** `POST /approvals/{placementId}/approve` is idempotent; a second call on an already-granted approval returns `200` without re-emitting the event.
- **Budget accounting:** remaining budget is computed from `paymentLedger.entries` at every `guardrailStep` call — no cached total. This prevents double-spend from concurrent workflow invocations on the same campaign (campaigns are single-workflow by design, but the check is stateless for correctness).
- **Stuck detection:** `StaleCampaignMonitor` ticks every 60 s; `CampaignFailedTimeout` covers both `EXECUTING` and `AWAITING_APPROVAL` states to prevent approvals from silently hanging indefinitely.
- **Sanitizer determinism:** `CredentialScrubber.scrub` is pure; no external state. The same input always yields the same scrubbed output.
