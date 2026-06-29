# User journeys — itsm-change-planner

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: change planned, approved, executed, and implemented

**Preconditions:** Service running on port 9979; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9979/`. App UI tab is visible.
2. In the form, enter summary "Upgrade nginx to 1.26.1 on web-prod-01", CI name "web-prod-01", category NORMAL. Click Submit.
3. A new change card appears with status `PLANNING`.
4. Within 30 s, status transitions to `AWAITING_CAB`.
5. In the CAB approval pane, enter reviewer name "j.smith" and click **Approve**.

**Expected:**
- Within 5 s of approval, status transitions to `APPROVED` then `EXECUTING` via SSE.
- The change progresses through each implementation step. After every step, a test step runs.
- Within ~5 minutes, status transitions to `IMPLEMENTED`.
- The expanded execution log shows one `StepRecord` per implementation step. Each record has `stepResult.ok = true` and `testResult.passed = true`.
- The `ChangeLedger` shows a non-empty `implementationPlan` (3–5 steps), a matching `testPlan`, and a `backoutPlan`.

## J2 — CAB rejection stops execution immediately

**Preconditions:** As J1.

**Steps:**
1. Submit any change request.
2. When status is `AWAITING_CAB`, enter reviewer name "cab-reviewer" and click **Reject** with comments "Risk too high for change window."

**Expected:**
- Status transitions to `REJECTED` via SSE.
- The `CabDecision` block on the expanded change shows `outcome = REJECTED`, the reviewer name, and the comments.
- No `StepExecuted` or `TestPassed` events appear on the entity (`GET /api/changes/{id}` returns an empty `executionLog`).
- The workflow ends; no further agent calls are dispatched.

## J3 — Test failure triggers automatic backout

**Preconditions:** As J1, with the mock LLM configured such that at least one `test-runner.json` entry has `passed=false`.

**Steps:**
1. Submit a change request and approve it via the CAB pane.
2. Observe the execution log as steps progress.

**Expected:**
- When a test step returns `passed=false`, status transitions to `ROLLING_BACK` via SSE.
- The execution log shows the failing `StepRecord` with `testResult.passed = false` and a non-empty `failureDetail`.
- The backout branch executes: `BackoutStepExecuted` entries appear on the entity, one per backout step, in reverse sequence order.
- After all backout steps, status transitions to `ROLLED_BACK`.
- The expanded change shows the partial execution log (steps up to the failure) and a set of backout entries.
- No further implementation steps are dispatched after the failing test.

## J4 — Production-touch guardrail blocks a disallowed CI

**Preconditions:** As J1.

**Steps:**
1. Submit a change request whose `ciName` is a valid seeded CI, but configure the mock planner (`planner.json`) to include an implementation step with `targetCi` set to a name NOT in `sample-data/ci-allowlist.json` (e.g., `core-router-01`).
2. Approve the change via the CAB pane.

**Expected:**
- When the executor loop reaches the disallowed step, the guardrail rejects it.
- A `StepBlocked` entry appears in the execution log with the blocker reason naming the disallowed CI.
- The `PlannerAgent` receives a `REVISE_STEP` task and returns a revised step targeting an allow-listed CI.
- The revised step passes the guardrail and proceeds to the `ExecutorAgent`.
- The original disallowed `targetCi` never appears in a `StepExecuted` event.

## J5 — Operator halt drains the current step pair

**Preconditions:** As J1.

**Steps:**
1. Submit and approve a change with multiple implementation steps.
2. While status is `EXECUTING`, click **Halt execution** in the operator pane with reason "emergency freeze."
3. Observe the in-flight step pair (execute + test) complete.

**Expected:**
- The in-flight `StepRecord` is recorded normally.
- The next loop iteration reads the halt flag, exits the loop, and emits `ChangeHaltedOperator`.
- Status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- Clicking **Resume** clears the flag; subsequent changes can start their workflows.
