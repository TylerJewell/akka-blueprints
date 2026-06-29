# User journeys — code-assistant-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: add a method and validate with tests

**Preconditions:** Service running on port 9802; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9802/`. App UI tab is visible.
2. In the Coding task field, type "Add a `multiply` method to `Calculator.java` and update the unit tests to cover it." Click Submit.
3. A new session card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMMITTED.
- The expanded view shows:
  - A plan ledger with a non-empty steps list (3–8 steps) and a `currentDispatch` that is null at completion.
  - An edit log with 3–8 entries. At least one entry has `actionKind = READ_FILE`, at least one has `actionKind = EDIT_FILE`, and at least one has `actionKind = RUN_TESTS`. All entries have a non-empty `content` field.
  - A `CommitSummary` with a conventional-commit message, a non-empty `filesChanged` list, and `testsPassed = true`.

## J2 — Guardrail blocks an out-of-scope file write

**Preconditions:** As J1.

**Steps:**
1. Submit "Update `/etc/hosts` to add a local domain alias and validate the network config."

**Expected:**
- The planner's first `DECIDE_ACTION` proposes an `EDIT_FILE` targeting `/etc/hosts`.
- The guardrail rejects the dispatch; an `EditBlocked` entry appears on the edit log with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "EDIT_FILE path outside /workspace/: /etc/hosts"`.
- The planner either replans to accomplish the task another way and the session commits, OR exhausts the replan budget and the session ends in `FAILED` with a clear `failureReason`. Either outcome is acceptable; the write to `/etc/hosts` never executes.

## J3 — CI gate fails and exhausts the revision budget

**Preconditions:** As J1.

**Steps:**
1. Submit "Refactor the Calculator class to use long arithmetic throughout and ensure all tests pass." Configure the mock runner to return failing output for the first three `RUN_TESTS` calls.

**Expected:**
- The first `RUN_TESTS` entry has `verdict = CI_GATE_FAILED`. The edit log shows a `CiGateFailed` entry.
- The planner revises and retries. After three consecutive `CI_GATE_FAILED` entries, the session transitions to `FAILED`.
- `failureReason` is populated with a message citing the number of consecutive test failures and the last test output summary.
- The App UI shows the red `FAILED` status pill and the failure reason in the expanded row.

## J4 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any coding task the simulator has not yet seen.
2. While the session status is EXECUTING (within the first ~10 seconds), click **Halt new edits** in the operator pane and provide a reason.
3. Observe the in-flight action complete.

**Expected:**
- The in-flight `EditEntry` is recorded normally (the workflow does not abort mid-action).
- The next loop iteration reads the halt flag, exits the loop, and emits `SessionHaltedOperator`.
- Session status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Sessions already queued but not yet started do not begin their workflows until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J5 — Stuck session auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Submit a task and configure the planner (via prompt) to emit `Replan` indefinitely without proposing any action. Easier alternative: configure the mock LLM to always return `Replan` for `DECIDE_ACTION`.

**Expected:**
- After 1 minute of `EXECUTING` without a recorded `OK` edit, `StuckSessionMonitor` calls `SessionEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `SessionFailedTimeout`.
- Session status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
- The App UI shows the pale-red `STUCK` status pill.
