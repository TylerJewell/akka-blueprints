# PLAN — month-end-closer

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
  classDef gate fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[CloseRunEndpoint]:::ep
  Entity[CloseRunEntity]:::ese
  WF[CloseRunWorkflow]:::wf
  Agent[CloseAgent]:::agent
  Gather[GatherTools]:::tool
  Validate[ValidateTools]:::tool
  Report[ReportTools]:::tool
  Gate[ApprovalGate]:::gate
  Scorer[ReconciliationScorer]:::gate
  View[CloseRunView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|gatherStep runSingleTask| Agent
  Agent -->|invokes| Gather
  Agent -->|invokes| Validate
  Agent -->|invokes| Report
  WF -->|approvalGate poll| Entity
  API -->|approveStep / rejectStep| Entity
  Entity -->|StepApproved / StepRejected| WF
  Agent -->|LedgerSnapshot / JournalEntrySet / ClosePackage| WF
  WF -->|recordSnapshot/EntrySet/Package| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|ReconciliationResult| WF
  WF -->|recordReconciliation| Entity
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
  participant Acc as Accounting User
  participant API as CloseRunEndpoint
  participant E as CloseRunEntity
  participant W as CloseRunWorkflow
  participant A as CloseAgent
  participant T as Tools (Gather/Validate/Report)
  participant Sc as ReconciliationScorer

  U->>API: POST /api/close-runs { entity, period }
  API->>E: create(entity, period)
  E-->>API: { closeRunId }
  API->>W: start(closeRunId, entity, period)
  W->>E: startGather
  W->>A: runSingleTask(GATHER_LEDGER_DATA, entity, period)
  A->>T: fetchLedgerLines + fetchChartOfAccounts
  T-->>A: List<LedgerLine> / ChartOfAccounts
  A-->>W: LedgerSnapshot
  W->>E: recordLedgerSnapshot
  W->>W: approvalGate(GATHER) — blocking
  Acc->>API: POST /close-runs/{id}/approve { step: GATHER }
  API->>E: approveStep(GATHER)
  E-->>W: StepApproved(GATHER)
  W->>E: startValidate
  W->>A: runSingleTask(VALIDATE_ENTRIES, ledgerSnapshot)
  A->>T: checkDebitCreditBalance + validateAccountCodes
  T-->>A: ValidationResult
  A-->>W: JournalEntrySet
  W->>E: recordJournalEntrySet
  W->>W: approvalGate(VALIDATE) — blocking
  Acc->>API: POST /close-runs/{id}/approve { step: VALIDATE }
  API->>E: approveStep(VALIDATE)
  E-->>W: StepApproved(VALIDATE)
  W->>E: startReport
  W->>A: runSingleTask(WRITE_CLOSE_REPORT, journalEntrySet)
  A->>T: formatTrialBalanceSummary + writeVarianceCommentary
  T-->>A: TrialBalanceSummary / String
  A-->>W: ClosePackage
  W->>E: recordClosePackage
  W->>Sc: score(closePackage, journalEntrySet, ledgerSnapshot)
  Sc-->>W: ReconciliationResult
  W->>E: recordReconciliation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `CloseRunEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> GATHERING: GatherStarted
  GATHERING --> AWAITING_GATHER_APPROVAL: LedgerDataGathered
  AWAITING_GATHER_APPROVAL --> VALIDATING: StepApproved(GATHER)
  AWAITING_GATHER_APPROVAL --> GATHERING: StepRejected(GATHER)
  VALIDATING --> AWAITING_VALIDATE_APPROVAL: EntriesValidated
  AWAITING_VALIDATE_APPROVAL --> REPORTING: StepApproved(VALIDATE)
  AWAITING_VALIDATE_APPROVAL --> VALIDATING: StepRejected(VALIDATE)
  REPORTING --> EVALUATED: ReconciliationScored
  GATHERING --> FAILED: CloseRunFailed
  VALIDATING --> FAILED: CloseRunFailed
  REPORTING --> FAILED: CloseRunFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

A `StepRejected` event does not transition to `FAILED` — it drives the workflow back into the previous task phase for a re-run, consuming another iteration of the agent's budget. Only an exhausted retry budget, a step timeout, or an explicit error transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  CloseRunEntity ||--o{ CloseRunCreated : emits
  CloseRunEntity ||--o{ GatherStarted : emits
  CloseRunEntity ||--o{ LedgerDataGathered : emits
  CloseRunEntity ||--o{ StepApproved : emits
  CloseRunEntity ||--o{ StepRejected : emits
  CloseRunEntity ||--o{ ValidateStarted : emits
  CloseRunEntity ||--o{ EntriesValidated : emits
  CloseRunEntity ||--o{ ReportStarted : emits
  CloseRunEntity ||--o{ CloseReportWritten : emits
  CloseRunEntity ||--o{ ReconciliationScored : emits
  CloseRunEntity ||--o{ CloseRunFailed : emits
  CloseRunView }o--|| CloseRunEntity : projects
  CloseRunWorkflow }o--|| CloseRunEntity : reads-and-writes
  CloseAgent ||--o{ LedgerSnapshot : returns
  CloseAgent ||--o{ JournalEntrySet : returns
  CloseAgent ||--o{ ClosePackage : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CloseRunEndpoint` | `api/CloseRunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CloseRunEntity` | `application/CloseRunEntity.java` (state in `domain/CloseRunRecord.java`, events in `domain/CloseRunEvent.java`) |
| `CloseRunWorkflow` | `application/CloseRunWorkflow.java` |
| `CloseAgent` | `application/CloseAgent.java` (tasks in `application/CloseTasks.java`) |
| `GatherTools` | `application/GatherTools.java` |
| `ValidateTools` | `application/ValidateTools.java` |
| `ReportTools` | `application/ReportTools.java` |
| `ApprovalGate` | `application/ApprovalGate.java` |
| `ReconciliationScorer` | `application/ReconciliationScorer.java` |
| `CloseRunView` | `application/CloseRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `gatherStep` 60 s, `validateStep` 60 s, `reportStep` 60 s, `evalStep` 5 s, `approvalGate(GATHER)` 3600 s, `approvalGate(VALIDATE)` 3600 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CloseRunWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4). The 3600 s on approval gates accommodates human review time.
- **Idempotency**: each workflow uses `"workflow-" + closeRunId` as the workflow id; restart of the same closeRunId is rejected by the workflow runtime. The agent instance id is `"agent-" + closeRunId` so each run has its own per-task conversation memory.
- **One agent per run**: `CloseAgent` runs three tasks per close run — GATHER, VALIDATE, REPORT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the agent room to recover from a tool call error and still complete the task.
- **Approval drives phase advance**: the workflow's `approvalGate` steps poll `CloseRunEntity.status`. The only way to advance past `AWAITING_GATHER_APPROVAL` is a `StepApproved(GATHER)` event written by `POST /api/close-runs/{id}/approve`. This is the application-level HITL gate — no timeout, no auto-advance. A `StepRejected` causes the workflow to replay the immediately-preceding task step.
- **Eval is synchronous and deterministic**: `ReconciliationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same close package always scores the same.
- **Task-boundary handoff is the dependency contract**: `gatherStep` writes `LedgerDataGathered` BEFORE the approval gate opens; `validateStep` reads the recorded `LedgerSnapshot` to build the VALIDATE task's instruction context; `reportStep` reads both. The agent is stateless across phases — it never holds gather + validate + report context in one conversation.
