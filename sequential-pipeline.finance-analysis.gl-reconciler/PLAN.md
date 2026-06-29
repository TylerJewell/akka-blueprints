# PLAN — gl-reconciler

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

  API[ReconciliationEndpoint]:::ep
  Entity[LedgerReconciliationEntity]:::ese
  WF[ReconciliationWorkflow]:::wf
  Agent[LedgerAgent]:::agent
  Fetch[FetchTools]:::tool
  Reconcile[ReconcileTools]:::tool
  Journal[JournalTools]:::tool
  Guard[PostingGuardrail]:::guard
  Checker[MaterialVarianceChecker]:::guard
  View[ReconciliationView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|fetchStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|invokes| Fetch
  Agent -->|invokes| Reconcile
  Agent -->|invokes| Journal
  Guard -->|recordValidationFailed| Entity
  Agent -->|LedgerSnapshot / ReconciliationResult / JournalEntry| WF
  WF -->|recordSnapshot/Reconciliation/Journal| Entity
  WF -->|reconcileStep check| Checker
  Checker -->|EscalationDecision| WF
  WF -->|raiseEscalation / approve / reject| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, no material variance)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant C as Controller (UI)
  participant API as ReconciliationEndpoint
  participant E as LedgerReconciliationEntity
  participant W as ReconciliationWorkflow
  participant A as LedgerAgent
  participant G as PostingGuardrail
  participant T as Tools (Fetch/Reconcile/Journal)
  participant MC as MaterialVarianceChecker

  C->>API: POST /api/reconciliations { accountSetId }
  API->>E: create(accountSetId)
  E-->>API: { runId }
  API->>W: start(runId, accountSetId)
  W->>E: startFetch
  W->>A: runSingleTask(FETCH_ENTRIES, accountSetId)
  A->>T: fetchLedgerEntries + fetchAccountBalances
  T-->>A: List<LedgerEntry> / List<AccountBalance>
  A-->>G: before-agent-response(LedgerSnapshot)
  G-->>A: accept
  A-->>W: LedgerSnapshot
  W->>E: recordSnapshot
  W->>A: runSingleTask(RECONCILE_ACCOUNTS, snapshot)
  A->>T: computeVariances + calculateNAV
  T-->>A: List<Variance> / NavCalculation
  A-->>G: before-agent-response(ReconciliationResult)
  G-->>A: accept
  A-->>W: ReconciliationResult
  W->>E: recordReconciliation
  W->>MC: check(result, threshold)
  MC-->>W: EscalationDecision(escalate=false)
  W->>A: runSingleTask(DRAFT_JOURNAL, reconciliation)
  A->>T: formatJournalLine + sumJournalLines
  T-->>A: JournalLine / JournalTotals
  A-->>G: before-agent-response(JournalEntry)
  G-->>A: accept (balanced, valid account codes)
  A-->>W: JournalEntry
  W->>E: recordJournal → ValidationPassed → JournalPosted
  E-.->>C: SSE event(POSTED)
```

## State machine — `LedgerReconciliationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> FETCHING: FetchStarted
  FETCHING --> FETCHED: EntriesFetched
  FETCHED --> RECONCILING: ReconcileStarted
  RECONCILING --> RECONCILED: ReconciliationCompleted
  RECONCILED --> PENDING_APPROVAL: EscalationRaised
  RECONCILED --> DRAFTING: draftStep (no escalation)
  PENDING_APPROVAL --> DRAFTING: EscalationApproved
  PENDING_APPROVAL --> REJECTED: EscalationRejected
  DRAFTING --> DRAFT_VALIDATED: ValidationPassed
  DRAFT_VALIDATED --> POSTED: JournalPosted
  FETCHING --> FAILED: RunFailed
  RECONCILING --> FAILED: RunFailed
  DRAFTING --> FAILED: RunFailed
  POSTED --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

`ValidationFailed` is a side-event recorded on the entity when `PostingGuardrail` rejects the draft; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  LedgerReconciliationEntity ||--o{ RunCreated : emits
  LedgerReconciliationEntity ||--o{ FetchStarted : emits
  LedgerReconciliationEntity ||--o{ EntriesFetched : emits
  LedgerReconciliationEntity ||--o{ ReconcileStarted : emits
  LedgerReconciliationEntity ||--o{ ReconciliationCompleted : emits
  LedgerReconciliationEntity ||--o{ EscalationRaised : emits
  LedgerReconciliationEntity ||--o{ EscalationApproved : emits
  LedgerReconciliationEntity ||--o{ EscalationRejected : emits
  LedgerReconciliationEntity ||--o{ DraftStarted : emits
  LedgerReconciliationEntity ||--o{ JournalDrafted : emits
  LedgerReconciliationEntity ||--o{ ValidationPassed : emits
  LedgerReconciliationEntity ||--o{ ValidationFailed : emits
  LedgerReconciliationEntity ||--o{ JournalPosted : emits
  LedgerReconciliationEntity ||--o{ RunFailed : emits
  ReconciliationView }o--|| LedgerReconciliationEntity : projects
  ReconciliationWorkflow }o--|| LedgerReconciliationEntity : reads-and-writes
  LedgerAgent ||--o{ LedgerSnapshot : returns
  LedgerAgent ||--o{ ReconciliationResult : returns
  LedgerAgent ||--o{ JournalEntry : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ReconciliationEndpoint` | `api/ReconciliationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `LedgerReconciliationEntity` | `application/LedgerReconciliationEntity.java` (state in `domain/ReconciliationRun.java`, events in `domain/ReconciliationEvent.java`) |
| `ReconciliationWorkflow` | `application/ReconciliationWorkflow.java` |
| `LedgerAgent` | `application/LedgerAgent.java` (tasks in `application/LedgerTasks.java`) |
| `FetchTools` | `application/FetchTools.java` |
| `ReconcileTools` | `application/ReconcileTools.java` |
| `JournalTools` | `application/JournalTools.java` |
| `PostingGuardrail` | `application/PostingGuardrail.java` |
| `MaterialVarianceChecker` | `application/MaterialVarianceChecker.java` |
| `ReconciliationView` | `application/ReconciliationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `fetchStep` 60 s, `reconcileStep` 60 s, `hitlStep` 72 h (controller window), `draftStep` 60 s, `validateStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ReconciliationWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4). The 72 h on `hitlStep` gives the controller a working-day window to review and decide.
- **Idempotency**: each workflow uses `"recon-" + runId` as the workflow id; restart of the same runId is rejected by the workflow runtime. The agent instance id is `"agent-" + runId` so each reconciliation run has its own per-task conversation memory.
- **One agent per run**: `LedgerAgent` runs three tasks per run — FETCH, RECONCILE, DRAFT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the posting guardrail room to reject an unbalanced draft and still let the agent self-correct.
- **Guardrail-driven retry**: when `PostingGuardrail` rejects a task result, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Material variance check is synchronous and deterministic**: `MaterialVarianceChecker` runs in-process inside `reconcileStep`. No LLM call, no external service. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `fetchStep` writes `EntriesFetched` BEFORE returning; `reconcileStep` reads the recorded `LedgerSnapshot` from the entity to build its task's instruction context; `draftStep` reads both `LedgerSnapshot` and `ReconciliationResult`. The agent itself is stateless across phases.
- **HITL is an explicit workflow state**: the `hitlStep` is not a notification side-channel — it is a genuine workflow step with a timer. The controller's `approve` / `reject` command is a command on the entity, which the workflow polls. Every decision is recorded in the entity log.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed run stays at the last successful event; the UI shows the partial state for the controller.
