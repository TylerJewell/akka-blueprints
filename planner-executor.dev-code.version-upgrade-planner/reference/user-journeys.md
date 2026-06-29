# User journeys — version-upgrade-planner

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path with approval gate

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9335/`. App UI tab is visible.
2. In the form, enter Source version `2.7.3`, Target version `2.10.0`, Requested by `engineer@example.com`. Click Submit.
3. A new job card appears with status `PLANNING`.

**Expected:**
- Within 5 s, status transitions to `EXECUTING` via SSE.
- The plan ledger shows 3–5 ordered phases; at least one has kind `MIGRATION` and the approval badge.
- When the first `MIGRATION` phase is about to dispatch, status transitions to `AWAITING_APPROVAL`. The approval pane appears with the phase name and rationale.
- The reviewer clicks `Approve`. Status returns to `EXECUTING` within 2 s via SSE.
- After all phases complete (within ~5 minutes), status transitions to `COMPLETED`.
- The expanded view shows:
  - A progress ledger with entries from `COMPAT_CHECKER`, `MIGRATION_APPLIER`, and `TEST_RUNNER`.
  - An `UpgradeReport` with a 60–120 word summary, a non-empty `appliedPhases` list, and `testsPassed: true`.

## J2 — Reviewer rejects a migration phase

**Preconditions:** As J1.

**Steps:**
1. Submit the same job as J1.
2. When the approval pane appears for the first `MIGRATION` phase, click `Reject` and enter a comment: "Not ready for db migration — rollback plan not reviewed yet."
3. Observe the job status.

**Expected:**
- The approval is resolved with `approved=false`.
- A `PhaseBlocked` progress entry appears with `verdict=BLOCKED_BY_APPROVAL` and the rejection comment as the blocker.
- The job status returns to `EXECUTING` as the planner receives the rejection and runs `DECIDE_PHASE`.
- The planner either emits `Replan` (adding the phase back after a review step, or substituting a safe alternative) or `Fail` if the replan budget is exhausted.
- In either outcome, the rejected `MIGRATION` phase never ran its migration script.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any job.
2. While the job status is `EXECUTING` (within the first 10 seconds of execution), click `Halt new phases` in the operator pane and provide a reason.
3. Observe the in-flight phase completes.

**Expected:**
- The in-flight `ProgressEntry` is recorded normally (the workflow does not abort mid-phase).
- The next loop iteration reads the halt flag in `checkHaltStep`, exits the loop, and emits `JobHaltedOperator`.
- Job status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- Clicking `Resume` clears the halt flag; new job submissions proceed normally.

## J4 — CI gate blocks next migration on test failure

**Preconditions:** As J1; the `test-runner.json` mock is configured to return a fixture with `failed > 0` on the first test run after a migration.

**Steps:**
1. Submit an upgrade job.
2. Allow the compatibility check to complete and approve the first migration phase.
3. After the `MIGRATION_APPLIER` records its progress entry, observe the `ciGateStep` triggering.

**Expected:**
- `TestRunnerAgent` returns a `PhaseResult` with `testReport.failed > 0` and at least one entry in `failingTests`.
- A `CiGateFailed` progress entry is recorded with `verdict=CI_GATE_FAILED` and the full `TestReport`.
- The job does **not** immediately dispatch the next migration phase.
- The planner receives the test report in its next `DECIDE_PHASE` call and emits `Replan`, adding a remediation or rollback phase. The job either recovers (runs the rollback, re-attempts the migration, passes tests on retry) or fails gracefully after the replan budget is exhausted.
- In no scenario does a subsequent migration phase run while the previous phase's test suite shows failures.
