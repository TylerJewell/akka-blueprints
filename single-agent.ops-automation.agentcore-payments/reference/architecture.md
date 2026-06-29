# Architecture: Agent-Initiated Payment Settlement

## Component graph

The component graph shows five components arranged around `PaymentTaskEntity`, which is the authoritative source of truth for all task state.

`PaymentTaskEndpoint` is the sole HTTP entry point. It accepts task submissions, routes approval decisions to `PaymentApprovalWorkflow`, and serves live ledger data via Server-Sent Events backed by `PaymentLedgerView`. The endpoint never mutates entity state directly — it sends commands and reads projections.

`PaymentAgentSession` is a stateless orchestrator. It receives the task description and spend parameters, decomposes the work into payment intents, and submits them to `PaymentTaskEntity` via tool calls. All durable state — spent amounts, payment records, task status — lives in the entity. The agent holds no state between tool calls.

`PaymentApprovalWorkflow` is started by `PaymentTaskEntity` when a payment intent exceeds the approval threshold. It owns the waiting loop: it holds the payment in `PENDING_APPROVAL` state, accepts a human decision from the endpoint, and calls back to the entity to finalize the outcome. The workflow also manages the approval timeout — if no decision arrives within the configured window, it emits a soft rejection so the agent can continue.

`PaymentLedgerView` is a key-value projection over `PaymentTaskEntity` events. It maintains one row per task with current spend, budget remaining, task status, and a count of pending approvals. The SSE stream in the endpoint subscribes to this projection directly — no polling, no timers.

---

## Interaction sequence — J1 happy path

The sequence diagram shows a task with two payment intents: one below threshold (auto-settled) and one above threshold (held for approval).

The critical pause points are:
1. **After `PaymentHeld`**: the agent suspends its iteration loop and the client's SSE stream carries the `PaymentHeld` event. The human reviewer is notified by an out-of-band mechanism (deployer responsibility) and submits a decision via `POST /tasks/{taskId}/approvals`.
2. **Workflow resolution**: `PaymentApprovalWorkflow` receives the decision and calls back to the entity. The entity emits `PaymentApproved` then `PaymentSettled`, and the agent resumes on its next `getTaskStatus` poll.

If the reviewer rejects the payment, the agent receives `REJECTED` from its next tool call and decides whether to substitute a lower-cost alternative.

---

## State machine — PaymentTaskEntity

The state machine has six stable states: `PENDING`, `ACTIVE`, `AWAITING_APPROVAL`, `BUDGET_EXCEEDED`, `COMPLETED`, and `FAILED`.

The `AWAITING_APPROVAL` state is entered any time a payment is held. The task can receive additional below-threshold payment settlements while one payment is awaiting approval — the entity tracks the held payment separately. The task returns to `ACTIVE` on each approval or rejection.

`BUDGET_EXCEEDED` is a terminal state entered from either `ACTIVE` or `AWAITING_APPROVAL`. No further payment intents are accepted. Any in-flight workflow is cancelled.

The CSS class overrides on the state diagram (Lesson 24) distinguish: `AWAITING_APPROVAL` (amber, requires attention), `BUDGET_EXCEEDED` (red, terminal halt), `COMPLETED` (teal, success), and `ACTIVE` (blue, normal operation).

---

## Entity model

Four structures participate in the data model:

`PaymentTask` is the root — one row per task with budget parameters and accumulated spend.

`PaymentRecord` is the line-item — one row per payment intent, keyed by `paymentId`, containing the amount, target API, and final status.

`PaymentApprovalWorkflow` is the durable workflow state — keyed by `paymentId`, containing the decision and expiry timestamp. At most one active workflow exists per task at any time.

`PaymentLedgerRow` is the read model — one projected row per task, updated on every relevant event, with pre-computed `remainingCents` and approval counts for fast read access.

---

## Governance layering

Two enforcement points operate independently:

**Layer 1 — HITL gate (H-001):** Evaluated synchronously inside `PaymentTaskEntity`'s command handler. The entity checks `amountCents >= approvalThresholdCents` before emitting any event. If true, it emits `PaymentHeld` and starts the workflow. The LLM never sees the approval decision — only the tool call result.

**Layer 2 — Budget cap (H-002):** Also evaluated synchronously in the entity command handler, before any `PaymentSettled` event. The cap is checked against settled spend only (rejected payments do not count). When the cap is breached, the entity emits `BudgetCapReached` and moves to `BUDGET_EXCEEDED` — the agent cannot override this by resubmitting.

Both layers are testable independently: H-001 by submitting an intent above threshold without a reviewer decision, H-002 by submitting a sequence of intents that sums to exactly the budget ceiling.
