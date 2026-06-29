# Architecture — finance-close-reconciler

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ReconciliationEndpoint` accepts a submission and writes a `PeriodSubmitted` event onto `PeriodEntity`. The `BalanceSanitizer` Consumer subscribes, masks confidential financial fields, and writes the sanitized balance back via `attachSanitized`. The same Consumer then starts a `ReconciliationWorkflow` instance. The workflow's `reconcileStep` calls `ReconciliationAgent` — the single AutonomousAgent — with the reconciliation rules as `TaskDef.instructions(...)` and the sanitized balance as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`WriteGuardrail`) validates each proposed GL write before it is recorded. Once a report passes, the workflow writes `ReportRecorded`, then pauses in `awaitSignOffStep` until the accountant calls the sign-off endpoint. After approval, the workflow runs `AttestationScorer` in `attestStep` and writes `AttestationCompleted`. `ReconciliationView` projects every entity event into a read-model row; `ReconciliationEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. `AttestationScorer` is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct pauses in the flow:

1. The `BalanceSanitizer` subscription lag between `PeriodSubmitted` and `BalanceSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `PeriodEntity` every 1 s up to its 15 s timeout.
3. The `awaitSignOffStep` pause — the workflow polls every 2 s with a 24 h timeout. This is the human-latency window. The accountant reviews the variance list at their own pace and calls the sign-off endpoint when ready.

The agent call itself is bounded by `reconcileStep`'s 90 s timeout, which accommodates both LLM latency and multiple guardrail-rejected tool-call retries. `attestStep` is synchronous and finishes in milliseconds.

## State machine

Eight states. The paths:

- The happy path is `SUBMITTED → SANITIZED → RECONCILING → REPORT_RECORDED → AWAITING_SIGNOFF → ATTESTED`.
- The rejected path branches at `AWAITING_SIGNOFF → REJECTED` when the accountant calls the sign-off endpoint with `decision: REJECTED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `RECONCILING`.
- There is no `POSTED` state. GL write proposals are stored on the report for a human operator to post; the blueprint stops at `ATTESTED`.

## Entity model

`PeriodEntity` is the source of truth. It emits nine event types. `ReconciliationView` projects every event into a row used by the UI. `BalanceSanitizer` subscribes to entity events to compute the sanitized form. `ReconciliationWorkflow` both reads (`getPeriod`) and writes (`markReconciling`, `recordReport`, `requestSignOff`, `grantSignOff`, `rejectSignOff`, `attest`, `fail`) on the entity. The relationship between `ReconciliationAgent` and `ReconciliationReport` is "returns" — the agent's task result is the report record.

## Defence-in-depth governance flow

For any report that lands in the entity log and reaches the audit trail, the trial balance passed through:

1. **Confidential-field sanitizer** — the model never sees margin percentages or internal entity codes; the audit log retains the raw form.
2. **ReconciliationAgent** — one model call, one structured output with tool calls.
3. **Before-tool-call guardrail** — invalid account codes, unbalanced entries, and closed-period writes are caught before the tool call executes. The ledger is never touched by a bad proposal.
4. **Human sign-off** — the accountant reads the variance list and approves before attestation. No report advances to the audit trail without explicit human approval.
5. **CI attestation gate** — the build fails if any fixture period lacks a complete event chain, catching regressions before promotion.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
