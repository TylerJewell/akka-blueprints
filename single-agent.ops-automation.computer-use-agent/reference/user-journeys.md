# User journeys — computer-use-agent

## J1 — Submit a form-filling task and watch it complete

**Preconditions:** Service running on declared port (`http://localhost:9391/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9391/` → App UI tab.
2. From the **Task template** dropdown, pick `fill-web-form`.
3. The task description pre-fills: "Open the contact form, fill in name, email, subject, then submit."
4. Enter a value in **Submitted by** and click **Run task**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s, then transitions to `RUNNING`.
- The action log in the right pane grows in real time. Each `ActionRecord` entry shows action type, target, and `ALLOWED` status.
- When the agent proposes `FORM_SUBMIT`, the card transitions to `AWAITING_CONFIRMATION` and the confirmation banner appears. The user clicks **Approve**.
- After approval, the action executes and the card returns to `RUNNING`. The `FORM_SUBMIT` entry updates to `CONFIRMED` in the action log.
- Within 60 s the task reaches `COMPLETED` with `OutcomeStatus.SUCCESS`. The outcome section shows the summary and stats (total actions ≥ 4, actionsConfirmed = 1, actionsBlocked = 0).

## J2 — Guardrail blocks a destructive action

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `execute-task.json` includes a `FILE_DELETE` entry.

**Steps:**
1. Submit any task using the `bulk-rename-files` template.
2. Watch the action log in the right pane; look for the `FILE_DELETE` entry.

**Expected:**
- The `FILE_DELETE` action appears in the log with status `BLOCKED` and `blockedReason = "BLOCKED_DESTRUCTIVE_ACTION: FILE_DELETE requires a prior ConfirmationReceived event."`.
- The task does NOT transition to `AWAITING_CONFIRMATION` for this action — the guardrail blocks it before the confirmation gate is reached.
- The agent receives the `BLOCKED` signal on its next turn and proposes an alternative (`FILE_READ` or a different path). The task continues.
- The final `TaskOutcome` shows `actionsBlocked ≥ 1`.

## J3 — High-impact action triggers the confirmation gate; user rejects

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the `fill-web-form` task.
2. Wait for the confirmation banner to appear (the agent has proposed `FORM_SUBMIT`).
3. Click **Reject** instead of **Approve**.

**Expected:**
- The `FORM_SUBMIT` entry in the action log shows status `REJECTED`.
- The task returns to `RUNNING`. The agent receives a `USER_REJECTED` signal on its next turn.
- The agent proposes an alternative action (e.g., `SCREENSHOT` to re-observe the state) or returns a `TaskOutcome{status: PARTIAL}` explaining that the form was not submitted.
- The entity never records an `ActionExecuted` entry with `status = ALLOWED` for the rejected `FORM_SUBMIT`.

## J4 — Operator halts a running task mid-execution

**Preconditions:** Service running. Any model provider. A long-running task is in `RUNNING` state (the `navigate-and-extract` template with multiple pages works well).

**Steps:**
1. Submit the `navigate-and-extract` task.
2. While the card shows `RUNNING` and the action log is growing, click the red **Halt** button on the card.

**Expected:**
- The card transitions to `HALTED` within 1–2 s.
- The action log stops updating. No new `ActionExecuted` events arrive after the halt.
- The service log shows no further agent invocations for this taskId after the halt event.
- `GET /api/tasks/{id}` returns `haltRequested: true`, `status: "HALTED"`, and the full `actionHistory` up to the halt moment. `outcome.status` is `"HALTED"`.

## J5 — Confirmation timeout transitions task to FAILED

**Preconditions:** Service running with mock LLM. Temporarily set the `awaitConfirmationStep` timeout to 10 s for this journey (or wait 5 min with the default).

**Steps:**
1. Submit the `fill-web-form` task.
2. When the confirmation banner appears, do not click Approve or Reject. Wait for the step timeout.

**Expected:**
- After the timeout elapses, the card transitions from `AWAITING_CONFIRMATION` to `FAILED`.
- The pending action does NOT appear in the action log as `ALLOWED` or `CONFIRMED` — it was never executed.
- `GET /api/tasks/{id}` returns `status: "FAILED"` and a `outcome` with `status: "FAILED"` and a summary that names confirmation timeout as the reason.
- The `actionHistory` is intact up to the `ConfirmationPending` event.

## J6 — Action log is complete and ordered after a full multi-step task

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the `navigate-and-extract` task (navigate to a product page, extract title and price).
2. Wait for `COMPLETED`.
3. Fetch `GET /api/tasks/{id}` and inspect `actionHistory`.

**Expected:**
- `actionHistory` is a non-empty list ordered by `executedAt` ascending.
- Every entry has a non-null `callId`, `actionType`, `targetSelector`, and `status`.
- `blockedReason` is `null` on every entry whose `status` is not `BLOCKED`.
- `totalActionsExecuted` in the outcome equals `actionHistory.size()`.
- No two entries share the same `callId`.
