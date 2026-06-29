# User journeys — self-discover-modules

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9914; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9914/`. App UI tab is visible.
2. In the Task prompt field, type "Explain why gradient descent can converge to a local minimum rather than a global minimum." Click Submit.
3. A new task card appears with status STRUCTURING.

**Expected:**
- Within 5 s, the Structurer selects modules and composes a structure; status remains STRUCTURING while the eval runs.
- The eval returns `passed=true`; status transitions to SOLVING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A `ReasoningStructure` with 3–6 adapted steps. At least one step has `role = DECOMPOSE` and at least one has `role = SYNTHESIZE`.
  - An `EvalResult` with `passed=true` and `score >= 0.7`.
  - A step execution timeline with one entry per module step, each with `ok=true` and a 3–6 sentence observation.
  - A `TaskAnswer` with a 60–120 word summary and 3–5 observation bullets, each prefixed by the module name.

## J2 — Eval rejects the first structure; Structurer revises

**Preconditions:** As J1, with the mock LLM configured to return a structure missing a SYNTHESIZE step on the first COMPOSE_STRUCTURE call and a valid structure on the first REVISE_STRUCTURE call.

**Steps:**
1. Submit "Analyse the trade-offs between consistency and availability in distributed databases."

**Expected:**
- The first `COMPOSE_STRUCTURE` call returns a structure that the eval scores below 0.7 (missing SYNTHESIZE step). A `StructureRejected` event appears on the task with `rejectionReason = "missing SYNTHESIZE step"`.
- The Structurer receives `rejectionReason` and produces a revised structure that includes a SYNTHESIZE step.
- The revised structure passes eval (`passed=true`); `StructureApproved` is emitted and status moves to SOLVING.
- The task completes normally with step executions and a final answer.
- The UI's expanded view shows `revisionNumber = 1` on the approved structure.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any task that requires 4 or more module steps (to give time to act).
2. While the task status is SOLVING (mid-way through step executions, within the first ~15 s), click **Halt new dispatches** in the operator pane and provide a reason ("reviewing output quality").
3. Observe the in-flight step execution finishes.

**Expected:**
- The current `StepExecution` is recorded normally — the workflow does not abort mid-step.
- The next loop iteration reads `halted=true` from `SystemControlEntity`, exits the solve loop, and emits `TaskHaltedOperator`.
- Task status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane reflects the `HALTED` state in real time via the `control-update` SSE event.
- Clicking **Resume** clears the flag; new task submissions proceed normally (previously halted tasks remain in `HALTED`).

## J4 — Revision budget exhausted; task fails

**Preconditions:** As J1, with the mock LLM configured to return a structure that fails eval on every COMPOSE_STRUCTURE and REVISE_STRUCTURE call (e.g., always returns fewer than 2 steps).

**Steps:**
1. Submit any task prompt.

**Expected:**
- The Structurer produces a structure that the eval rejects.
- The workflow calls REVISE_STRUCTURE; the revised structure also fails eval (revisionNumber = 1).
- A second revision is attempted; the second revised structure also fails eval (revisionNumber = 2).
- The workflow transitions to `failStep` without a third Structurer call.
- `TaskFailed` is emitted; task status moves to `FAILED`.
- `failureReason` reads exactly `"structure eval failed after maximum revisions"`.
- The UI's expanded view shows the last rejected structure and its eval score.
