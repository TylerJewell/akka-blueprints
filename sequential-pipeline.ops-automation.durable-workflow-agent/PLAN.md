# PLAN — durable-workflow-backed-agent

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
  classDef sched fill:#1a1a2e,stroke:#a78bfa,color:#a78bfa;

  API[RemediationEndpoint]:::ep
  IncEntity[IncidentEntity]:::ese
  DREntity[DurabilityReportEntity]:::ese
  WF[RemediationWorkflow]:::wf
  Agent[OpsAgent]:::agent
  Diag[DiagnoseTools]:::tool
  Rem[RemediateTools]:::tool
  Ver[VerifyTools]:::tool
  Guard[BudgetGuardrail]:::guard
  Eval[DurabilityEvaluator]:::sched
  Timer[DurabilityTimerAction]:::sched
  View[IncidentView]:::view
  App[AppEndpoint]:::ep

  API -->|create| IncEntity
  API -->|start| WF
  WF -->|diagnoseStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Diag
  Agent -->|invokes| Rem
  Agent -->|invokes| Ver
  Guard -->|recordBudgetExhausted| IncEntity
  Agent -->|DiagnosisReport / RemediationPlan / VerificationResult| WF
  WF -->|recordDiagnosis/Remediation/Verification| IncEntity
  IncEntity -.->|projects| View
  Timer -->|hourly fire| Eval
  API -->|trigger| Eval
  Eval -->|queries| View
  Eval -->|record| DREntity
  API -->|latest report| DREntity
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as RemediationEndpoint
  participant E as IncidentEntity
  participant W as RemediationWorkflow
  participant A as OpsAgent
  participant G as BudgetGuardrail
  participant T as Tools (Diagnose/Remediate/Verify)

  U->>API: POST /api/incidents { alertId, serviceId }
  API->>E: create(alertId, serviceId)
  E-->>API: { incidentId }
  API->>W: start(incidentId, alertId, serviceId)
  W->>E: startDiagnose
  W->>A: runSingleTask(DIAGNOSE_INCIDENT, context)
  A->>G: before-tool-call(queryMetrics, DIAGNOSE)
  G-->>A: accept (count=1, cap=8)
  A->>T: queryMetrics + fetchLogs
  T-->>A: List<MetricSample> / List<LogEntry>
  A-->>W: DiagnosisReport
  W->>E: recordDiagnosis
  W->>E: startRemediate
  W->>A: runSingleTask(APPLY_REMEDIATION, context)
  A->>G: before-tool-call(applyAction, REMEDIATE)
  G-->>A: accept (count=1, cap=6)
  A->>T: applyAction + confirmAction
  T-->>A: ActionReceipt / ActionStatus
  A-->>W: RemediationPlan
  W->>E: recordRemediation
  W->>E: startVerify
  W->>A: runSingleTask(VERIFY_FIX, context)
  A->>G: before-tool-call(checkHealth, VERIFY)
  G-->>A: accept (count=1, cap=6)
  A->>T: checkHealth + confirmResolved
  T-->>A: HealthCheck / ResolutionCheck
  A-->>W: VerificationResult
  W->>E: recordVerification
  E-.->>U: SSE event(VERIFIED)
```

## State machine — `IncidentEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> DIAGNOSING: DiagnoseStarted
  DIAGNOSING --> DIAGNOSED: DiagnosisCompleted
  DIAGNOSED --> REMEDIATING: RemediateStarted
  REMEDIATING --> REMEDIATED: RemediationCompleted
  REMEDIATED --> VERIFYING: VerifyStarted
  VERIFYING --> VERIFIED: VerificationCompleted
  DIAGNOSING --> FAILED: IncidentFailed
  REMEDIATING --> FAILED: IncidentFailed
  VERIFYING --> FAILED: IncidentFailed
  VERIFIED --> [*]
  FAILED --> [*]
```

`BudgetExhausted` is a side-event recorded on the entity for audit; it does not change the incident status — the agent's accumulated result is still used by the workflow step if the agent can return a partial typed output. Only an exhausted step-recovery budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  IncidentEntity ||--o{ IncidentCreated : emits
  IncidentEntity ||--o{ DiagnoseStarted : emits
  IncidentEntity ||--o{ DiagnosisCompleted : emits
  IncidentEntity ||--o{ RemediateStarted : emits
  IncidentEntity ||--o{ RemediationCompleted : emits
  IncidentEntity ||--o{ VerifyStarted : emits
  IncidentEntity ||--o{ VerificationCompleted : emits
  IncidentEntity ||--o{ BudgetExhausted : emits
  IncidentEntity ||--o{ IncidentFailed : emits
  DurabilityReportEntity ||--o{ DurabilityReportRecorded : emits
  IncidentView }o--|| IncidentEntity : projects
  RemediationWorkflow }o--|| IncidentEntity : reads-and-writes
  DurabilityEvaluator }o--|| IncidentView : queries
  DurabilityEvaluator ||--o{ DurabilityReport : produces
  OpsAgent ||--o{ DiagnosisReport : returns
  OpsAgent ||--o{ RemediationPlan : returns
  OpsAgent ||--o{ VerificationResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RemediationEndpoint` | `api/RemediationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `IncidentEntity` | `application/IncidentEntity.java` (state in `domain/IncidentRecord.java`, events in `domain/IncidentEvent.java`) |
| `DurabilityReportEntity` | `application/DurabilityReportEntity.java` |
| `RemediationWorkflow` | `application/RemediationWorkflow.java` |
| `OpsAgent` | `application/OpsAgent.java` (tasks in `application/OpsTasks.java`) |
| `DiagnoseTools` | `application/DiagnoseTools.java` |
| `RemediateTools` | `application/RemediateTools.java` |
| `VerifyTools` | `application/VerifyTools.java` |
| `BudgetGuardrail` | `application/BudgetGuardrail.java` |
| `DurabilityEvaluator` | `application/DurabilityEvaluator.java` |
| `DurabilityTimerAction` | `application/DurabilityTimerAction.java` |
| `IncidentView` | `application/IncidentView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `diagnoseStep` 90 s, `remediateStep` 90 s, `verifyStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(RemediationWorkflow::error)`. The 90 s on agent-calling steps accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"remediation-" + incidentId` as the workflow id; restart of the same incidentId is rejected by the workflow runtime. The agent instance id is `"agent-" + incidentId` so each incident has its own per-task conversation memory.
- **One agent per incident**: `OpsAgent` runs three tasks per incident — DIAGNOSE, REMEDIATE, VERIFY — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the budget guardrail room to fire and still allow the agent to return a partial result.
- **Budget-cap halt**: when `BudgetGuardrail` halts a tool call, the halt is returned as a structured error to the agent loop. The agent cannot issue further tool calls in this task; it must return with what it has. If the accumulated result does not satisfy the typed output, the step's default recovery retries up to 2 times; if all retries fail, the workflow transitions to `FAILED`.
- **Scheduled eval is asynchronous**: `DurabilityTimerAction` fires independently of any incident lifecycle. The manual trigger endpoint (`POST /api/durability/trigger`) fires the same `DurabilityEvaluator.evaluate` logic immediately, making the evaluation testable without an hourly wait.
- **Task-boundary handoff is the dependency contract**: `diagnoseStep` writes `DiagnosisCompleted` BEFORE returning; `remediateStep` reads the recorded `DiagnosisReport` from the entity to build its task's instruction context; `verifyStep` reads both `DiagnosisReport` and `RemediationPlan`. The agent itself is stateless across phases — it never holds diagnose + remediate + verify context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed incident stays at the last successful event; the UI shows the partial state.
