# User journeys — planner-executor-general-planner-executor

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9712; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9712/`. App UI tab is visible.
2. In the Goal field, type "Draft and send a weekly status report to the team." Click Submit.
3. A new plan card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A plan ledger with a non-empty `openSteps` list (3–8 items at creation), all items moved to `completedSteps` at completion.
  - A step log with 3–8 entries. Every entry has a non-empty `result` and `verdict = OK`.
  - A `PlanOutcome` with a 60–100 word summary and a `completedSteps` list matching the step log.

## J2 — Guardrail blocks an out-of-policy step

**Preconditions:** As J1.

**Steps:**
1. Submit the goal "Clean up the workspace by running `rm -rf /tmp/*` and report what was freed."

**Expected:**
- The planner's first `DECIDE_STEP` proposes a step whose text starts with `rm`.
- The guardrail rejects the decision; a `StepBlocked` entry appears on the step log with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "destructive command not permitted: rm"`.
- The planner either revises the plan to use a different approach and the plan completes, OR exhausts the replan budget and the plan ends in `FAILED` with a clear `failureReason`. Either outcome is acceptable; what matters is that the destructive command never reaches `ExecutorAgent`.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any goal the simulator has not yet seen.
2. While the plan status is EXECUTING (within the first ~10 seconds), click **Halt new steps** in the operator pane and provide a reason.
3. Observe the in-flight step completes.

**Expected:**
- The in-flight `StepEntry` is recorded normally — the workflow does not abort mid-step.
- The next loop iteration reads the halt flag, exits the loop, and emits `PlanHaltedOperator`.
- Plan status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- Clicking **Resume** clears the flag; new goals submitted after Resume start normally.

## J4 — Replan budget exhaustion forces failure

**Preconditions:** As J1, with `model-provider = mock` and the mock planner fixture configured to emit `Replan` on three consecutive `DECIDE_STEP` calls.

**Steps:**
1. Submit a goal whose mock fixture sequence is: Replan → Replan → Replan.

**Expected:**
- First `Replan`: `PlanRevised` event is emitted; plan ledger is updated; loop continues.
- Second `Replan`: another `PlanRevised` event is emitted; replan counter reaches 2.
- Third `Replan`: the workflow does not call the Planner for a fourth replan. Instead, it transitions directly to `failStep`.
- `PlanFailed` is emitted with `failureReason` containing "replan budget exhausted".
- Plan status moves to `FAILED`. The step log shows the two completed revision cycles and no further step entries after the second Replan.
