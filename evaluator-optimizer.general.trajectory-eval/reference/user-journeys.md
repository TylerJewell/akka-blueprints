# User journeys — trajectory-eval

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9235; valid model-provider API key set, OR mock mode active. A reference path for scenario `kb-refund-routing` is registered.

**Steps:**
1. Open `http://localhost:9235/`. App UI tab is visible.
2. Select scenario `kb-refund-routing` from the dropdown. Leave the task description blank (uses the stored default). Click Submit.
3. A new evaluation card appears with status `RUNNING`.

**Expected:**
- The evaluation transitions to `RUNNING` immediately; the first trajectory attempt appears in the expanded view.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's verdict is `PASS` and the evaluation transitions to `PASSED`, OR
  - Attempt 1's verdict is `FAIL` with one or more deviations listed, the evaluation transitions back to `RUNNING`, and attempt 2 appears shortly after; this continues until `PASS` or the retry ceiling.
- On `PASSED`, the terminal block shows the matched trajectory and "matched on attempt N."
- The expanded view shows every attempt's trajectory steps, guardrail verdict, evaluator verdict, deviation count, and full deviation report.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus a scenario configured to always produce diverging trajectories (submit the literal scenario id `"test-force-fail"`, which the mock provider's seedFor logic always answers with a `FAIL` verdict containing two deviations).

**Steps:**
1. Submit scenario `"test-force-fail"` with the default settings.

**Expected:**
- Evaluation progresses `RUNNING` → `EVALUATING` → `RUNNING` → `EVALUATING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `FAIL`, the evaluation transitions to `FAILED_FINAL` (not stuck in `EVALUATING`).
- The terminal block shows the attempt with the fewest deviations as the "closest match" and the `failureReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its trajectory steps, guardrail OK verdict, FAIL verdict, deviation count, and deviation report.
- `GET /api/evaluations/{id}` returns the full Evaluation with all 4 attempts in `attempts[]` and `status: "FAILED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit a scenario configured in mock mode to produce a 23-step trajectory on the first run (exceeds the default 20-step ceiling).

**Expected:**
- Attempt 1's trajectory has `stepCount = 23`. The guardrail records `verdict.passed = false`, `reasonCode = "OVER_STEP_LIMIT"`, with detail naming the excess.
- The Evaluator is NOT called for attempt 1. The evaluation stays in `RUNNING`.
- The Runner is called again with structured deviation feedback. Attempt 2 produces a trajectory at or below the step ceiling and the guardrail passes.
- The evaluator scores attempt 2 normally. The loop continues until `PASS` or the retry ceiling.
- The expanded view shows attempt 1 with the over-limit trajectory and the red `OVER_STEP_LIMIT` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one evaluation has completed (any terminal state).

**Steps:**
1. Click the evaluation card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per evaluated attempt, with `verdict`, `deviationCount`, and `stepLimitExceeded` populated.
- The terminal transition (EvaluationPassed or EvaluationFailedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/evaluations/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — Reference path management

**Preconditions:** Service running on port 9235.

**Steps:**
1. Call `POST /api/references` with body `{ "scenarioId": "my-scenario", "steps": [{"toolName": "lookup"}, {"toolName": "respond"}] }`.
2. Call `GET /api/references/my-scenario`.
3. Submit scenario `"my-scenario"` via `POST /api/evaluations`.

**Expected:**
- `POST /api/references` returns `200 { "scenarioId": "my-scenario" }`.
- `GET /api/references/my-scenario` returns the stored `ReferencePath` with `registeredAt` populated and `updatedAt` null.
- `POST /api/evaluations` accepts the scenario and returns a valid `evaluationId`; submitting an unknown scenario id returns `404`.
