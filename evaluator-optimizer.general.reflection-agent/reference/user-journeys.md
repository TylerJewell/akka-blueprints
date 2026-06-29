# User journeys — reflection-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9360; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9360/`. App UI tab is visible.
2. In the Task prompt field, type "Explain what an event-sourced entity is and when you would use it." Click Submit.
3. A new task card appears with status `GENERATING`.

**Expected:**
- Within 1 s, status transitions to `GENERATING` (already there) and the first iteration's response appears.
- Within 60 s of submission, either:
  - Iteration 1's reflection is `ACCEPT` and the task transitions to `ACCEPTED`, OR
  - Iteration 1's reflection is `REVISE` with 1–3 change requests and the task transitions to `REFLECTING`, with iteration 2 appearing shortly after; this continues until either an `ACCEPT` or the ceiling.
- On `ACCEPTED`, the terminal block shows the accepted response and "best of N iterations."
- The expanded view shows every iteration's response, reflector verdict, score, and notes.

## J2 — Halt at iteration ceiling

**Preconditions:** As J1, plus an override that forces the Reflector to always return `REVISE` (test mode — submit the literal prompt `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the task `"test-force-reject"`.

**Expected:**
- Task progresses `GENERATING` → `REFLECTING` → `GENERATING` → `REFLECTING` → … for `maxIterations` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the task transitions to `REJECTED_FINAL` (not stuck in `REFLECTING`).
- The terminal block shows the highest-scoring iteration's response as "best of 4 iterations" and the `rejectionReason` reads `"max iterations reached (4)"`.
- All 4 iterations are present in the expanded view, each with its response, REVISE verdict, and score.
- `GET /api/tasks/{id}` returns the full Task with all 4 iterations in `iterations[]` and `status: "REJECTED_FINAL"`.

## J3 — Eval-event timeline

**Preconditions:** At least one task has completed (any terminal state).

**Steps:**
1. Click the task card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per reflected iteration, with `verdict` and `score` populated.
- The terminal transition (TaskAccepted or TaskRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/tasks/{id}` includes the eval event data (either as a separate `evalEvents[]` field or embedded in iterations). The UI does not require a separate fetch to render the timeline.

## J4 — Revision fidelity

**Preconditions:** A task has been submitted and iteration 1 returned `REVISE` with at least two change requests.

**Steps:**
1. Expand the task card that progressed through a revision cycle.
2. Compare the change requests in iteration 1's reflection with the response in iteration 2.

**Expected:**
- Iteration 2's response visibly addresses each change request from iteration 1 (e.g., if the note says "opening paragraph lacks a clear thesis," the second response starts with an explicit thesis statement).
- The reflector's verdict on iteration 2 reflects the improvements (score should be equal to or higher than iteration 1's score).
- Both the critique notes and the revised response are visible in the expanded view without any additional clicks.
