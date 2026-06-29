# PLAN — secops-triage

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[FindingEndpoint]:::ep
  FindingE[FindingEntity]:::ese
  ApprovalE[ApprovalEntity]:::ese
  Enricher[FindingEnricher]:::cons
  WF[TriageWorkflow]:::wf
  DriftWF[DriftCheckWorkflow]:::wf
  Agent[VulnerabilityTriageAgent]:::agent
  Guard[RemediationGuardrail]:::guard
  Drift[DriftEvaluator]:::guard
  View[FindingView]:::view
  App[AppEndpoint]:::ep

  API -->|ingest| FindingE
  API -->|approve/reject| ApprovalE
  FindingE -.->|FindingIngested| Enricher
  Enricher -->|attachEnriched| FindingE
  Enricher -->|start workflow| WF
  WF -->|awaitEnrichedStep poll| FindingE
  WF -->|triageStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|block + createApproval| ApprovalE
  Guard -->|requestApproval| FindingE
  Agent -->|TriageVerdict| WF
  WF -->|recordVerdict| FindingE
  WF -->|approvalGateStep poll| ApprovalE
  WF -->|markRemediated| FindingE
  DriftWF -->|driftCheckStep score| Drift
  Drift -->|reads recent| View
  Drift -->|DriftAlertRaised| FindingE
  FindingE -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path — critical finding, approved)

```mermaid
sequenceDiagram
  autonumber
  participant U as Analyst (UI)
  participant API as FindingEndpoint
  participant FE as FindingEntity
  participant AE as ApprovalEntity
  participant En as FindingEnricher
  participant W as TriageWorkflow
  participant A as VulnerabilityTriageAgent
  participant G as RemediationGuardrail

  U->>API: POST /api/findings
  API->>FE: ingest(raw)
  FE-->>API: { findingId }
  FE-.->>En: FindingIngested
  En->>En: enrich (asset context + threat intel)
  En->>FE: attachEnriched
  En->>W: start(findingId)
  W->>FE: poll getFinding
  FE-->>W: enriched.isPresent()
  W->>FE: triageStarted
  W->>A: runSingleTask(attachment: finding.json)
  A->>G: before-tool-call(patch-deploy)
  G->>AE: createApproval
  G->>FE: requestApproval
  G-->>A: block(approval-required)
  A-->>W: TriageVerdict(CRITICAL_IMMEDIATE)
  W->>FE: recordVerdict
  W->>W: approvalGateStep poll
  U->>API: POST /findings/{id}/approve
  API->>AE: grant(analystId, reason)
  AE-->>W: ApprovalGranted
  W->>FE: markRemediated
  FE-.->>U: SSE event(REMEDIATED)
```

## State machine — `FindingEntity`

```mermaid
stateDiagram-v2
  [*] --> INGESTED
  INGESTED --> ENRICHED: FindingEnriched
  ENRICHED --> TRIAGING: TriageStarted
  TRIAGING --> VERDICT_RECORDED: VerdictRecorded
  VERDICT_RECORDED --> PENDING_APPROVAL: ApprovalRequested (CRITICAL/HIGH)
  VERDICT_RECORDED --> MONITORED: FindingMonitored (MEDIUM)
  VERDICT_RECORDED --> ACCEPTED: FindingAccepted (LOW)
  PENDING_APPROVAL --> REMEDIATED: RemediationApproved
  PENDING_APPROVAL --> REMEDIATION_REJECTED: RemediationRejected
  TRIAGING --> FAILED: FindingFailed (agent error)
  INGESTED --> FAILED: FindingFailed (enricher error)
  REMEDIATED --> [*]
  REMEDIATION_REJECTED --> [*]
  MONITORED --> [*]
  ACCEPTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  FindingEntity ||--o{ FindingIngested : emits
  FindingEntity ||--o{ FindingEnriched : emits
  FindingEntity ||--o{ TriageStarted : emits
  FindingEntity ||--o{ VerdictRecorded : emits
  FindingEntity ||--o{ ApprovalRequested : emits
  FindingEntity ||--o{ RemediationApproved : emits
  FindingEntity ||--o{ RemediationRejected : emits
  FindingEntity ||--o{ DriftAlertRaised : emits
  FindingEntity ||--o{ FindingFailed : emits
  ApprovalEntity ||--o{ ApprovalCreated : emits
  ApprovalEntity ||--o{ ApprovalGranted : emits
  ApprovalEntity ||--o{ ApprovalRejected : emits
  FindingView }o--|| FindingEntity : projects
  FindingEnricher }o--|| FindingEntity : subscribes
  TriageWorkflow }o--|| FindingEntity : reads-and-writes
  TriageWorkflow }o--|| ApprovalEntity : reads-and-writes
  VulnerabilityTriageAgent ||--o{ TriageVerdict : returns
  DriftEvaluator }o--|| FindingView : reads
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `FindingEndpoint` | `api/FindingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `FindingEntity` | `application/FindingEntity.java` (state in `domain/Finding.java`, events in `domain/FindingEvent.java`) |
| `ApprovalEntity` | `application/ApprovalEntity.java` (state in `domain/ApprovalDecision.java`, events in `domain/ApprovalEvent.java`) |
| `FindingEnricher` | `application/FindingEnricher.java` |
| `TriageWorkflow` | `application/TriageWorkflow.java` |
| `DriftCheckWorkflow` | `application/DriftCheckWorkflow.java` |
| `VulnerabilityTriageAgent` | `application/VulnerabilityTriageAgent.java` (tasks in `application/TriageTasks.java`) |
| `RemediationGuardrail` | `application/RemediationGuardrail.java` |
| `DriftEvaluator` | `application/DriftEvaluator.java` |
| `FindingView` | `application/FindingView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitEnrichedStep` 15 s, `triageStep` 60 s, `approvalGateStep` 1800 s, `remediationStep` 5 s, `driftCheckStep` 30 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TriageWorkflow::error)`. The 60 s on `triageStep` accommodates LLM latency (Lesson 4).
- **Approval gate**: `approvalGateStep` polls `ApprovalEntity` every 2 s with a 30-minute step timeout. If the analyst does not respond within 30 minutes, the workflow fails over to `error` and the finding lands in `FAILED`.
- **Idempotency**: every workflow uses `"triage-" + findingId` as the workflow id. `FindingEnricher` Consumer is allowed to redeliver `FindingIngested` because `FindingEntity.attachEnriched` is event-version-guarded — a second enrich attempt against an already-enriched finding is a no-op.
- **One agent per finding**: AutonomousAgent instance id is `"triage-" + findingId`. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-driven iterations.
- **Drift check is isolated**: `DriftCheckWorkflow` uses a stable id `"drift-monitor"` and operates on `FindingView` data only. It does not write to individual `FindingEntity` instances; it emits onto a sentinel entity id `"drift-monitor"` so the drift alert is queryable.
- **No saga / no compensation**: finding enrichment and verdict recording are append-only event writes. There is nothing external to roll back.
