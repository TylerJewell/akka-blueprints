# User journeys — project-tracker

## J1 — New task arrives and gets assigned

**Preconditions:** Service running on port 9822; valid model-provider API key set (or mock LLM selected); `PlannerPoller` enabled.

**Steps:**
1. Open `http://localhost:9822/` → App UI tab.
2. Wait up to 20 s for the first simulated task.

**Expected:**
- Task appears with status CREATED.
- Within 30 s the status transitions to ASSIGNED. The assignment card shows the recommended owner name, rationale, and confidence.
- The task's `currentOwnerName` field is populated.
- The `PlannerWorkflow` for that task is visible in the Architecture tab's component count.

## J2 — Overdue task receives a nudge

**Preconditions:** A task in ASSIGNED or IN_PROGRESS whose `dueDate` is in the past (the simulator seeds at least one such task).

**Steps:**
1. Open App UI tab. Identify the OVERDUE task.
2. Wait up to 5 minutes for `StaleTaskChecker` to tick.

**Expected:**
- Task status transitions to NUDGED.
- `nudgeCount` increments to 1. `lastNudgedAt` is populated.
- The nudge tone chip shows `FRIENDLY`.
- The detail pane shows the drafted message text.
- No actual outbound network call leaves the process (the `sendNudge` tool is a stub).

## J3 — Three nudges trigger escalation

**Preconditions:** A task in NUDGED with `nudgeCount == 2`.

**Steps:**
1. Wait for `StaleTaskChecker` to fire again (or reduce the check interval to 30 s for testing via `STALE_CHECKER_SECONDS=30`).

**Expected:**
- The task's `nudgeCount` reaches 3 and status transitions to ESCALATED.
- The nudge tone is `ESCALATORY`.
- A `TaskEscalated` event is written to `PlannerTaskEntity`.
- The task card shows a red ESCALATED pill.

## J4 — Complete a task and receive an eval score

**Preconditions:** A task in ASSIGNED, IN_PROGRESS, or NUDGED. Reduce eval interval for testing via `EVAL_RUNNER_SECONDS=60`.

**Steps:**
1. Select the task in the list.
2. Click Complete.
3. Wait up to 60 s.

**Expected:**
- Status transitions to COMPLETED immediately. `completedAt` is populated.
- Within 60 s the card shows an eval score chip (1–5) and the rationale is visible in the detail pane.
- `GET /api/board/{id}` returns `evalScore` populated.

## J5 — Guardrail blocks an assignment to a full owner

**Preconditions:** The simulator's `board-events.jsonl` contains at least one task that will be recommended to the owner whose id ends in `_FULL`.

**Steps:**
1. Watch the App UI for a task that stays in CREATED longer than expected.
2. Check the service logs for a `DispatchResult` with `dispatched=false`.

**Expected:**
- The task stays in CREATED status. No `TaskAssigned` event is emitted.
- The log entry contains a `blockReason` naming the capacity issue.
- The task is re-eligible for assignment on the next `PlannerWorkflow` run (the workflow ends cleanly; a new one can be triggered by a board update).

## J6 — Board audit trail is complete

**Preconditions:** Service has been running for at least 2 minutes with default intervals.

**Steps:**
1. Call `GET /api/board` and note all task ids.
2. For one completed task, call `GET /api/board/{id}`.

**Expected:**
- The response includes `assignment.rationale`, `nudgeCount`, `evalScore`, and `evalRationale` (or null where not yet reached).
- The `createdAt` and `completedAt` timestamps are present and consistent with observed transitions.
