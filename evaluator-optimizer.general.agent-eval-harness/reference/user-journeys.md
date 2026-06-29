# User journeys — agent-eval-harness

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Passing run on the default suite

**Preconditions:** Service running on port 9796; valid model-provider API key set, OR mock mode active. Default suite registered (seeded by `RunScheduler` on first tick).

**Steps:**
1. Open `http://localhost:9796/`. App UI tab is visible.
2. Select "default" in the Suite dropdown. Leave the threshold at 0.8. Click Trigger Run.
3. A new run card appears with status `PENDING`.

**Expected:**
- Within 1 s, status transitions to `RUNNING` and the first case's response begins appearing.
- Each case card populates incrementally as `CaseEvaluated` events stream via SSE — the user sees progress case by case, not a single flush at the end.
- When all cases are processed and `passRate >= 0.8`, the run transitions to `PASSED`.
- The terminal block shows the final pass rate and "N of M cases passed."
- The expanded view shows every case's raw output, judge verdict pill, confidence score badge, and judge notes.
- `GET /api/runs/{id}` returns the full `EvalRun` with all case results in `results[]` and `status: "PASSED"`.

## J2 — CI gate failure when accuracy falls below threshold

**Preconditions:** As J1.

**Steps:**
1. Register a suite with all cases designed to produce `FAIL` judgments: `POST /api/suites` with `suiteName = "always-fail"`, 3 cases whose expected answers do not match what the evaluator returns, `defaultPassingThreshold = 0.9`.
2. Trigger a run against "always-fail" with threshold 0.9.

**Expected:**
- All 3 cases result in `FAIL` judgments; `passRate = 0.0`.
- After `aggregateStep`, the workflow emits `GateFailed` with `{ passRate: 0.0, threshold: 0.9, failCount: 3 }`.
- The run transitions to `FAILED` (not stuck in `RUNNING`).
- The terminal block shows the `failureReason` string: `"accuracy gate: passRate=0.0 below threshold=0.9 (failCount=3)"`.
- All 3 case results are preserved in the expanded view, each with a red `FAIL` pill and the judge's bullets.
- No `RunPassed` event appears in the entity's event log for this run.
- `GET /api/runs/{id}` returns `status: "FAILED"` and all 3 case results.

## J3 — Incremental SSE progress during a multi-case run

**Preconditions:** As J1. Use a suite with at least 4 cases.

**Steps:**
1. Trigger a run against a 4-case suite.
2. Immediately open the App UI and watch the live run card without refreshing.

**Expected:**
- After `RunStarted`, the case counter reads `0/4 evaluated`.
- After each `CaseEvaluated` event, the counter increments: `1/4`, `2/4`, `3/4`, `4/4`.
- The per-case timeline row for each case appears as soon as its `CaseEvaluated` event arrives — the next case is not yet visible.
- No full-page refresh is required at any point; the SSE stream delivers all updates.
- The run transitions to a terminal state only after the `4/4` counter is shown.

## J4 — Accuracy snapshot timeline

**Preconditions:** At least one run has completed (any terminal state).

**Steps:**
1. Click the completed run card to expand.
2. Scroll to the terminal block.

**Expected:**
- The terminal block includes an `AccuracySnapshot` section showing `passRate`, `passCount`, `failCount`, and `suiteName`.
- `GET /api/runs/{id}` includes the accuracy figures in the run response; the UI does not require a separate fetch to render the terminal block.
- If the `AccuracySampler` has already ticked since the run completed, no duplicate snapshot is visible — the sampler's idempotency check prevented a second write.
