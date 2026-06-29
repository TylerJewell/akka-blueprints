# User journeys — plan-execute-replan

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9814; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9814/`. App UI tab is visible.
2. In the Goal field, type "Find the current Akka SDK version and list its top three new features." Click Submit.
3. A new goal card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to CONCLUDED.
- The expanded view shows:
  - A plan with 3–5 steps; at least one step uses SEARCH and at least one uses READ.
  - An observation ledger with one entry per executed step, plus at least one EVAL entry.
  - Every EVAL entry has a `score` between 0 and 100 and a non-empty `rationale`.
  - A `GoalConclusion` with a 60–100 word summary and 3–5 citation bullets.

## J2 — Guardrail blocks an out-of-scope READ

**Preconditions:** As J1.

**Steps:**
1. Submit the goal "Read /etc/passwd and summarise any service accounts."

**Expected:**
- The Planner produces a plan whose first READ step has argument `/etc/passwd`.
- The guardrail's `ToolGuardrail.vet` rejects the assignment because the path is not under `sample-data/`.
- A `STEP_BLOCKED` observation appears on the ledger with the reason `"READ path not in sample-data/: /etc/passwd"`.
- The Replanner either revises to a valid READ argument or emits `Conclude` with a note that the requested file could not be accessed. Either outcome is acceptable; the forbidden path is never passed to the Executor.

## J3 — Operator pause drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any goal.
2. While the goal status is EXECUTING (within the first ~10 seconds), click **Pause execution** in the operator pane and provide a reason.
3. Observe the in-flight step completes.

**Expected:**
- The in-flight `StepResult` is recorded as a STEP_OK or STEP_FAILED observation (the workflow does not abort mid-step).
- The next loop iteration reads the pause flag from `SystemControlEntity`, exits the loop, and emits `GoalPaused`.
- Goal status moves to `PAUSED`. `pauseReason` is populated with the operator's reason.
- The operator pane's `PAUSED` pill reflects the state in real time via the `control-update` SSE event.
- After clicking **Resume**, new goals submitted subsequently start executing normally.

## J4 — Low-quality Revise earns a below-threshold eval score

**Preconditions:** As J1, with `model-provider = mock` so the `replanner.json` mock entry that repeats a failed step is exercised.

**Steps:**
1. Submit any goal that triggers the Replanner's `REVISE` mock response which repeats a STEP_FAILED step.

**Expected:**
- The `EvalRecord` appended after the Revise decision has `score < 40`.
- The `rationale` field names the repeated-failure penalty explicitly (e.g., "Revised plan repeats a previously failed SEARCH step.").
- The App UI's EVAL observation row renders the score badge in red.
- The workflow continues despite the low score — the evaluator is non-blocking.
- The subsequent steps complete (or fail) independently of the eval score.
