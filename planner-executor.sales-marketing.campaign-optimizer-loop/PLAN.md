# PLAN — campaign-optimizer-loop

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

  Planner[PlannerAgent]:::agent
  Copy[CopyWriterAgent]:::agent
  Audience[AudienceSegmenterAgent]:::agent
  Publish[AssetPublisherAgent]:::agent
  Perf[PerformanceAnalystAgent]:::agent

  WF[CampaignWorkflow]:::wf
  Camp[CampaignEntity]:::ese
  Appr[ApprovalEntity]:::ese
  Queue[RequestQueue]:::ese
  View[CampaignView]:::view
  Consumer[CampaignRequestConsumer]:::cons
  Sim[CampaignSimulator]:::ta
  PerfMon[PerformanceMonitor]:::ta
  API[CampaignEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|approve/reject| Appr
  Sim -.->|every 120s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN / DECIDE / COMPOSE_REPORT| Planner
  WF -->|WRITE_COPY| Copy
  WF -->|SELECT_AUDIENCE| Audience
  WF -->|PUBLISH_ASSETS| Publish
  WF -->|ANALYZE_PERFORMANCE| Perf
  WF -->|emit events| Camp
  WF -->|poll| Appr
  Camp -.->|projects| View
  API -->|query/SSE| View
  PerfMon -.->|every 60s| Camp
```

## Interaction sequence — J1 (happy path, approval granted)

```mermaid
sequenceDiagram
  autonumber
  participant U as User / Marketer
  participant API as CampaignEndpoint
  participant Q as RequestQueue
  participant C as CampaignRequestConsumer
  participant W as CampaignWorkflow
  participant P as PlannerAgent
  participant S as Specialist (Copy/Audience/Publish/Perf)
  participant E as CampaignEntity
  participant A as ApprovalEntity
  participant V as CampaignView

  U->>API: POST /api/campaigns {goal, targetAudience, channels}
  API->>Q: append CampaignSubmitted
  API-->>U: 202 {campaignId}
  Q->>C: CampaignSubmitted
  C->>W: start({campaignId, goal})
  W->>E: emit CampaignCreated (PLANNING)
  W->>P: PLAN_CAMPAIGN(brief)
  P-->>W: CampaignLedger
  W->>E: emit CampaignPlanned, status EXECUTING
  loop until SubmitForApproval | Fail
    W->>P: DECIDE(ledgers)
    P-->>W: Continue(DispatchDecision)
    W->>W: guardrail.check(stepResult) [copy only]
    W->>S: runSingleTask(step)
    S-->>W: StepResult
    W->>E: emit StepRecorded (RunEntry)
  end
  W->>E: emit ApprovalRequested, status AWAITING_APPROVAL
  W->>A: poll until decision
  U->>API: POST /api/campaigns/{id}/approve {decidedBy, note}
  API->>A: grant(decidedBy, note)
  A-->>W: ApprovalGranted
  W->>E: emit CampaignApproved, CampaignWentLive, status LIVE
  W->>P: COMPOSE_REPORT
  P-->>W: CampaignReport
  W->>E: emit CampaignCompleted, status COMPLETED
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `CampaignEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: CampaignPlanned
  EXECUTING --> EXECUTING: StepRecorded / StepBlocked / LedgerRevised
  EXECUTING --> AWAITING_APPROVAL: ApprovalRequested
  AWAITING_APPROVAL --> LIVE: CampaignApproved
  AWAITING_APPROVAL --> REJECTED: CampaignRejected
  LIVE --> LIVE: PerformanceAlertFired / StepRecorded
  LIVE --> COMPLETED: CampaignCompleted
  EXECUTING --> FAILED: CampaignFailed
  LIVE --> FAILED: CampaignFailed
  REJECTED --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  CampaignEntity ||--o{ CampaignPlanned : emits
  CampaignEntity ||--o{ StepDispatched : emits
  CampaignEntity ||--o{ StepBlocked : emits
  CampaignEntity ||--o{ StepRecorded : emits
  CampaignEntity ||--o{ LedgerRevised : emits
  CampaignEntity ||--o{ ApprovalRequested : emits
  CampaignEntity ||--o{ CampaignApproved : emits
  CampaignEntity ||--o{ CampaignRejected : emits
  CampaignEntity ||--o{ CampaignWentLive : emits
  CampaignEntity ||--o{ PerformanceAlertFired : emits
  CampaignEntity ||--o{ CampaignCompleted : emits
  CampaignEntity ||--o{ CampaignFailed : emits
  CampaignView }o--|| CampaignEntity : projects
  ApprovalEntity ||--o{ ApprovalGranted : emits
  ApprovalEntity ||--o{ ApprovalDenied : emits
  RequestQueue ||--o{ CampaignSubmitted : emits
  CampaignRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `CopyWriterAgent` | `application/CopyWriterAgent.java` |
| `AudienceSegmenterAgent` | `application/AudienceSegmenterAgent.java` |
| `AssetPublisherAgent` | `application/AssetPublisherAgent.java` |
| `PerformanceAnalystAgent` | `application/PerformanceAnalystAgent.java` |
| `CampaignWorkflow` | `application/CampaignWorkflow.java` |
| `CampaignEntity` | `application/CampaignEntity.java` (state in `domain/Campaign.java`, events in `domain/CampaignEvent.java`) |
| `ApprovalEntity` | `application/ApprovalEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `CampaignView` | `application/CampaignView.java` |
| `CampaignRequestConsumer` | `application/CampaignRequestConsumer.java` |
| `CampaignSimulator` | `application/CampaignSimulator.java` |
| `PerformanceMonitor` | `application/PerformanceMonitor.java` |
| `CopyGuardrail` | `application/CopyGuardrail.java` |
| `KpiThresholds` | `application/KpiThresholds.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `SpecialistTasks` | `application/SpecialistTasks.java` |
| `CampaignEndpoint` | `api/CampaignEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s, `decideStep` 45 s, `composeReportStep` 60 s, `approvalGateStep` 1800 s (30-minute human window).
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` becomes `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same `(specialist, step)` pair at most three times; a fourth attempt becomes `Fail`.
- **Approval gate:** `approvalGateStep` polls `ApprovalEntity.get` with a 1-second yield loop; after 30 minutes without a decision, the workflow emits `CampaignFailed` with `failureReason = "approval timeout"`.
- **Performance monitor race:** `PerformanceMonitor` emits `PerformanceAlertFired` directly on `CampaignEntity`; the workflow re-reads the entity at the top of its loop and detects the alert via `campaign.status == LIVE && pendingAlert`. The monitor is idempotent — it only fires once per 60 s window regardless of how many KPIs miss.
- **Idempotency:** `CampaignEndpoint.submit` deduplicates on `(goal, requestedBy)` over a 10 s window.
- **Guardrail scope:** `CopyGuardrail.check` runs only on `StepResult` from `CopyWriterAgent`. All other specialist results flow directly to `recordStep`.
