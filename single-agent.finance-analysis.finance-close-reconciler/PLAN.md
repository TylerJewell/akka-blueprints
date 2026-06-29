# PLAN — finance-close-reconciler

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

  API[ReconciliationEndpoint]:::ep
  Entity[PeriodEntity]:::ese
  Sanitizer[BalanceSanitizer]:::cons
  WF[ReconciliationWorkflow]:::wf
  Agent[ReconciliationAgent]:::agent
  Guard[WriteGuardrail]:::guard
  Scorer[AttestationScorer]:::guard
  View[ReconciliationView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|PeriodSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|reconcileStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|ReconciliationReport| WF
  WF -->|recordReport| Entity
  WF -->|awaitSignOffStep poll| Entity
  API -->|POST signoff| Entity
  WF -->|attestStep score| Scorer
  Scorer -->|AttestationResult| WF
  WF -->|attest| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Accountant (UI)
  participant API as ReconciliationEndpoint
  participant E as PeriodEntity
  participant S as BalanceSanitizer
  participant W as ReconciliationWorkflow
  participant A as ReconciliationAgent
  participant G as WriteGuardrail
  participant Sc as AttestationScorer

  U->>API: POST /api/periods
  API->>E: submit(submission)
  E-->>API: { periodId }
  E-.->>S: PeriodSubmitted
  S->>S: mask confidential fields
  S->>E: attachSanitized
  S->>W: start(periodId)
  W->>E: poll getPeriod
  E-->>W: sanitized.isPresent()
  W->>E: markReconciling
  W->>A: runSingleTask(rules + attachment)
  A->>G: before-tool-call(proposed GL write)
  G-->>A: accept
  A-->>W: ReconciliationReport
  W->>E: recordReport(report)
  W->>E: requestSignOff
  W->>E: poll awaitSignOff
  U->>API: POST /api/periods/{id}/signoff APPROVED
  API->>E: grantSignOff
  E-->>W: signOff.isPresent() + APPROVED
  W->>Sc: score(period, rules)
  Sc-->>W: AttestationResult
  W->>E: attest(attestation)
  E-.->>U: SSE event(ATTESTED)
```

## State machine — `PeriodEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: BalanceSanitized
  SANITIZED --> RECONCILING: ReconciliationStarted
  RECONCILING --> REPORT_RECORDED: ReportRecorded
  REPORT_RECORDED --> AWAITING_SIGNOFF: SignOffRequested
  AWAITING_SIGNOFF --> ATTESTED: SignOffGranted + AttestationCompleted
  AWAITING_SIGNOFF --> REJECTED: SignOffRejected
  RECONCILING --> FAILED: PeriodFailed (agent error)
  SUBMITTED --> FAILED: PeriodFailed (sanitizer error)
  ATTESTED --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PeriodEntity ||--o{ PeriodSubmitted : emits
  PeriodEntity ||--o{ BalanceSanitized : emits
  PeriodEntity ||--o{ ReconciliationStarted : emits
  PeriodEntity ||--o{ ReportRecorded : emits
  PeriodEntity ||--o{ SignOffRequested : emits
  PeriodEntity ||--o{ SignOffGranted : emits
  PeriodEntity ||--o{ SignOffRejected : emits
  PeriodEntity ||--o{ AttestationCompleted : emits
  PeriodEntity ||--o{ PeriodFailed : emits
  ReconciliationView }o--|| PeriodEntity : projects
  BalanceSanitizer }o--|| PeriodEntity : subscribes
  ReconciliationWorkflow }o--|| PeriodEntity : reads-and-writes
  ReconciliationAgent ||--o{ ReconciliationReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ReconciliationEndpoint` | `api/ReconciliationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PeriodEntity` | `application/PeriodEntity.java` (state in `domain/Period.java`, events in `domain/PeriodEvent.java`) |
| `BalanceSanitizer` | `application/BalanceSanitizer.java` |
| `ReconciliationWorkflow` | `application/ReconciliationWorkflow.java` |
| `ReconciliationAgent` | `application/ReconciliationAgent.java` (tasks in `application/ReconciliationTasks.java`) |
| `WriteGuardrail` | `application/WriteGuardrail.java` |
| `AttestationScorer` | `application/AttestationScorer.java` |
| `AttestationGateTest` | `test/AttestationGateTest.java` |
| `ReconciliationView` | `application/ReconciliationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `reconcileStep` 90 s, `awaitSignOffStep` 86400 s (24 h — human latency is unbounded), `attestStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ReconciliationWorkflow::error)`. The 90 s on `reconcileStep` accommodates both LLM latency and multiple guardrail-rejected tool-call retries (Lesson 4).
- **Idempotency**: every workflow uses `"recon-" + periodId` as the workflow id; the `BalanceSanitizer` Consumer is allowed to redeliver `PeriodSubmitted` events because `PeriodEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized period is a no-op.
- **One agent per period**: the AutonomousAgent instance id is `"reconciler-" + periodId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail-driven retry**: when `WriteGuardrail` rejects a candidate tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow's `reconcileStep` fails over to `error` and the entity transitions to `FAILED`.
- **HITL pause**: `awaitSignOffStep` uses a 24 h timeout. If the accountant does not respond within 24 h, the step times out and the workflow transitions to `FAILED` with reason `"sign-off-timeout"`.
- **Attestation is synchronous and deterministic**: `AttestationScorer` runs in-process inside `attestStep`. No LLM call — the same event chain always produces the same attestation result.
- **No saga / no compensation**: GL writes proposed by the agent are tool-call outputs stored in `ReconciliationReport.proposedGlWrites`. Actual posting to an ERP is out of scope; the blueprint records the proposals so a human operator can post them after sign-off.
