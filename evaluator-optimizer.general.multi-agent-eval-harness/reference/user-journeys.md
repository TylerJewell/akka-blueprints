# User journeys — multi-agent-eval-harness

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Passing run on first attempt

**Preconditions:** Service running on port 9784; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9784/`. App UI tab is visible.
2. In the Suite name field, type `reasoning-baseline`. Leave the pass threshold at the default 0.75. Click Submit.
3. A new run card appears with status `RUNNING`.

**Expected:**
- Within 2 s, the run card shows `RUNNING` and the scenario list begins populating.
- Within 90 s of submission, the orchestrator returns an `AggregateResult` with `aggregateScore >= 0.75`.
- The run transitions to `JUDGING`; the judge returns a `PASS` verdict.
- The run transitions to `PASSED`; the terminal block shows the final score and "run accepted."
- `ciGateBlocked` is `false` on `GET /api/runs/{id}`.
- The expanded view shows every scenario result with its `thresholdMet` value and the judgment with `overallRationale`.

## J2 — Failing run exhausts retries

**Preconditions:** As J1, plus mock mode seeded so the judge always returns `FAIL` (submit the suite name `test-force-fail`, which the mock provider always answers with `FAIL`).

**Steps:**
1. Submit the suite `test-force-fail` with pass threshold 0.75.

**Expected:**
- Run progresses `RUNNING` → `JUDGING` → `RUNNING` (retry 1) → `JUDGING` (retry 1) → `RUNNING` (retry 2) → `JUDGING` (retry 2) → `FAILED`.
- After the 2nd retry ends in `FAIL`, the run transitions to `FAILED` (not stuck in `JUDGING`).
- The terminal block shows the best-scoring judgment's rationale and `failureReason = "max retries reached (2)"`.
- All three judgment attempts are present in the expanded view.
- `GET /api/runs/{id}` returns the full run with `status: "FAILED"`, `judgments` containing 3 entries, and all scenario results intact.

## J3 — CI gate fires on a below-threshold pass

**Preconditions:** As J1.

**Steps:**
1. Submit a suite with `passThreshold` set to `0.95` (above what the mock orchestrator will return).

**Expected:**
- The orchestrator returns an `AggregateResult` with `aggregateScore` in the range `[0.80, 0.90]`.
- The judge may return `PASS` on its own rubric (e.g., score = 0.88), but `finalScore < passThreshold`.
- A `CIGateBlocked` event is emitted; `ciGateBlocked = true` on the run.
- The App UI shows a red "CI gate blocked" banner on the run card, even though `status = "PASSED"`.
- `GET /api/runs/{id}` returns `ciGateBlocked: true` and `finalScore: 0.88` alongside `status: "PASSED"`.

## J4 — Eval-event timeline populated

**Preconditions:** At least one run has reached a terminal state (any outcome).

**Steps:**
1. Click any completed run card to expand.

**Expected:**
- The timeline shows one `ScenarioEvalRecorded` event per judged scenario, with `scenarioId`, `verdict`, `score`, and `thresholdMet` populated.
- The terminal transition (`RunPassed` or `RunFailed`) is also surfaced as a final `ScenarioEvalRecorded` event carrying the loop-level `aggregateScore` and `ciGateBlocked` flag.
- `GET /api/runs/{id}` includes the scenario results and judgment data in the response body. The UI does not require a separate fetch to render the timeline.
- Within 60 s of the run completing, the `EvalSampler` has recorded events for all judged scenarios (visible as `ScenarioEvalRecorded` events in the entity's event log).
