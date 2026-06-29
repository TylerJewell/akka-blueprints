# Architecture — month-end-closer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence, with human approval gates between the first two handoffs. `CloseRunEndpoint` accepts a `{entity, period}` POST, writes `CloseRunCreated` onto `CloseRunEntity`, and starts `CloseRunWorkflow` keyed by `"workflow-" + closeRunId`. The workflow's first step (`gatherStep`) emits `GatherStarted`, then calls `CloseAgent` with `TaskDef.taskType(GATHER_LEDGER_DATA)` and a `phase = GATHER` metadata tag. The agent invokes `GatherTools.fetchLedgerLines` and `GatherTools.fetchChartOfAccounts`. Once the agent returns a `LedgerSnapshot`, the workflow writes `LedgerDataGathered` onto the entity and enters the `approvalGate(GATHER)` step, which blocks until an accounting user calls `POST /api/close-runs/{id}/approve`. On approval, the workflow advances to `validateStep` — same pattern, the VALIDATE task carries `phase = VALIDATE`. After `EntriesValidated` lands, the workflow enters `approvalGate(VALIDATE)`, which blocks until a second approval. On approval, `reportStep` runs with `phase = REPORT`. After `CloseReportWritten` lands, `evalStep` runs `ReconciliationScorer` over the recorded `(LedgerSnapshot, JournalEntrySet, ClosePackage)` triple — no LLM call — and writes `ReconciliationScored`. `CloseRunView` projects every event into a read-model row; `CloseRunEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `ReconciliationScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example with an application-level HITL gate.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `gatherStep` and `validateStep`, the workflow writes `LedgerDataGathered` onto the entity. The next step then reads `ledgerSnapshot` from the entity to build the VALIDATE task's instruction context. The agent never sees gather-phase context inside the validate task's conversation; the typed handoff is the only path information travels.
2. The HITL gates are structurally enforced. The workflow's `approvalGate` step has no timer — it blocks indefinitely until a human decision arrives. This is not a soft check; the next phase cannot start without the `StepApproved` event on the entity.
3. Every approval and rejection is durable. `StepApproved` and `StepRejected` events are written onto `CloseRunEntity` with the approver identity and timestamp. A close audit that asks "who approved the gather phase for ACME-US 2026-05 and when?" has a queryable answer in the event log.

The agent calls themselves are bounded by per-step timeouts (60 s on gather / validate / report). The approval gates carry 3600 s timeouts to accommodate human review. `evalStep` is synchronous and finishes in milliseconds.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → GATHERING → AWAITING_GATHER_APPROVAL → VALIDATING → AWAITING_VALIDATE_APPROVAL → REPORTING → EVALUATED`.
- A rejection from the gather approval loops back: `AWAITING_GATHER_APPROVAL → GATHERING` — the agent reruns the GATHER task and the output is re-presented for approval.
- A rejection from the validate approval loops back: `AWAITING_VALIDATE_APPROVAL → VALIDATING` — same pattern.
- Three failure transitions land in `FAILED`: an agent error during `GATHERING`, `VALIDATING`, or `REPORTING`. A `FAILED` close run's prior data is preserved on the entity — the UI shows the partial state.

There is no `FILED` or `PUBLISHED` state. The close package is advisory output; the accounting team files it in their GL system outside this service. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`CloseRunEntity` is the source of truth. It emits eleven event types — three lifecycle starts, three lifecycle completions (ledger data, entries, report), the reconciliation score, the approval and rejection gates, and the failure. `CloseRunView` projects every event into a row used by the UI. `CloseRunWorkflow` both reads (`getCloseRun`) and writes (`startGather`, `recordLedgerSnapshot`, `approveStep`, `rejectStep`, `startValidate`, `recordJournalEntrySet`, `startReport`, `recordClosePackage`, `recordReconciliation`, `fail`) on the entity. The relationship between `CloseAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any close package that lands in the entity log, the data passed through:

1. **HITL approval gates** — after GATHER and after VALIDATE, an accounting user must explicitly approve before the next phase starts. The entity records who approved, what the decision was, and when.
2. **CloseAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every emitted close package gets a 1–5 reconciliation score. Debit/credit balance, account-code validity, accrual reversal completeness, and variance within threshold are each worth one point on a base of 1.

Each layer is independent. The HITL gate does not check mathematical balance; the evaluator does not check approval history. Removing one layer opens an explicit gap the other does not silently cover.
