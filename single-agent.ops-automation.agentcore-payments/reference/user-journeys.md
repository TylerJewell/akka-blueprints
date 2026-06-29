# User Journeys: Agent-Initiated Payment Settlement

## J1 — Task with mixed-threshold payments settles in order

**Setup:** Submit a task with `budgetCents=5000`, `approvalThresholdCents=1000`, and a description that requires three API calls estimated at 400¢, 1200¢, and 600¢ respectively.

**Steps:**
1. `POST /tasks` — expect `201` with `status: PENDING`.
2. Within 5 seconds, `GET /tasks/task-001` — expect `status: ACTIVE`.
3. Subscribe to `GET /tasks/task-001/stream`.
4. Receive `payment-held` SSE event for the 1200¢ payment.
5. `POST /tasks/task-001/approvals` with `APPROVE`.
6. Receive `payment-approved` then `payment-settled` SSE events.
7. Receive `task-completed` event.

**Acceptance criteria:**
- Payment order in the entity matches submission order.
- The 400¢ and 600¢ payments have `approvalRequired: false` and `status: SETTLED`.
- The 1200¢ payment has `approvalRequired: true`, `status: SETTLED`, and `reviewedBy` set.
- `accumulatedSpentCents` equals 2200.

---

## J2 — Budget cap halts the agent mid-task

**Setup:** Submit a task with `budgetCents=2000`, `approvalThresholdCents=5000` (threshold above budget so no approvals are needed), and a description that would require five payments of 500¢ each.

**Steps:**
1. `POST /tasks` — expect `201`.
2. Watch SSE stream.
3. Observe three `payment-settled` events (total 1500¢).
4. Observe the fourth intent (500¢) causes `accumulatedSpentCents + 500 = 2000 >= 2000` — budget cap reached.
5. Receive `budget-cap-reached` SSE event.
6. `GET /tasks/task-001` — expect `status: BUDGET_EXCEEDED`.

**Acceptance criteria:**
- Exactly three payments with `status: SETTLED`.
- The fourth payment intent is not recorded as `SETTLED`; entity emitted `BudgetCapReached` instead.
- No fifth payment intent was submitted.
- `GET /tasks` shows `remainingCents: 0`.

---

## J3 — Reviewer rejects a held payment; agent substitutes

**Setup:** Submit a task with `budgetCents=5000`, `approvalThresholdCents=500`, and a description that requires two API calls at 800¢ and 300¢.

**Steps:**
1. `POST /tasks`.
2. Receive `payment-held` for the 800¢ payment.
3. `POST /tasks/task-001/approvals` with `REJECT` and `note: "Exceeds per-call policy"`.
4. Receive `payment-rejected` SSE event.
5. Agent proceeds — it submits a lower-cost alternative at 300¢.
6. Both 300¢ payments settle.
7. Receive `task-completed`.

**Acceptance criteria:**
- Rejected payment has `status: REJECTED`, `reviewedBy` set, and `amountCents: 800`.
- Rejected payment amount does NOT appear in `accumulatedSpentCents`.
- Two 300¢ payments are `SETTLED`; `accumulatedSpentCents = 600`.

---

## J4 — Approval timeout triggers soft rejection

**Setup:** Submit a task with `budgetCents=5000`, `approvalThresholdCents=100` (very low, so most payments are held), and configure `approvalTimeoutSeconds=5` for this test.

**Steps:**
1. `POST /tasks`.
2. Receive `payment-held` for the first payment.
3. Wait 6 seconds without posting any approval.
4. Receive `payment-rejected` SSE event (timeout-triggered).
5. `GET /tasks/task-001` — expect `status: ACTIVE` (not `FAILED`).

**Acceptance criteria:**
- Payment moves from `PENDING_APPROVAL` to status `REJECTED` with no `reviewedBy`.
- Task status remains `ACTIVE` — the timeout is a soft rejection, not a task failure.
- Agent may continue submitting further intents.

---

## J5 — Concurrent approval and budget-cap interaction

**Setup:** Submit a task with `budgetCents=2000`, `approvalThresholdCents=800`. Two payments of 900¢ each would individually require approval but together would exceed the budget.

**Steps:**
1. `POST /tasks`.
2. First intent (900¢) is held for approval.
3. While approval is pending, the agent submits a second intent (900¢).
4. Entity evaluates: `accumulatedSpentCents(0) + 900 = 900 < 2000` — second intent also held.
5. Approve first payment.
6. Entity settles first payment: `accumulatedSpentCents = 900`.
7. Approve second payment.
8. Entity evaluates: `900 + 900 = 1800 < 2000` — settles.
9. Task completes with `accumulatedSpentCents = 1800`.

**Acceptance criteria:**
- Both payments settle after independent approvals.
- At no point does `accumulatedSpentCents` exceed `budgetCents`.
- `pendingApprovals` in the ledger view correctly reflects 2 then 1 then 0 as approvals land.

---

## J6 — SSE stream delivers all events without gaps

**Setup:** Submit a task that produces at least five events (created, agent started, two payments, completed).

**Steps:**
1. Open SSE stream before submitting the task.
2. `POST /tasks`.
3. Collect all SSE events until `task-completed`.
4. Compare event sequence against `GET /tasks/{taskId}` payment history.

**Acceptance criteria:**
- Every event visible in the entity state is also present in the SSE stream.
- No events are duplicated in the stream.
- `task-completed` is the final event; no further events arrive after it.
- Reconnecting the SSE stream after `task-completed` serves the final state immediately (no wait).
