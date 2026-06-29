# User journeys — ragas-eval-harness

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9873; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9873/`. App UI tab is visible.
2. In the Question field, type "What is the default retry ceiling in the evaluation harness?". Leave Corpus tag empty. Click Submit.
3. A new run card appears with status `ANSWERING`.

**Expected:**
- Within 1 s, the first attempt's answer appears in the expanded view.
- The grounding verdict pill on attempt 1 reads `OK`.
- Within 90 s of submission, either:
  - Attempt 1's RAGAS score is `PASS` and the run transitions to `PASSED`, OR
  - Attempt 1's score is `RETRY` with `failedMetrics` listed and the run transitions back to `ANSWERING`, with attempt 2 appearing shortly after; this continues until `PASS` or the retry ceiling.
- On `PASSED`, the terminal block shows the accepted answer and "passed on attempt N."
- The expanded view shows every attempt's answer, grounding verdict, per-metric scores, evaluator verdict, and feedback.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the evaluator to always return `RETRY` (test mode — submit the literal question `"test-force-fail"`, which the mock provider's seedFor logic always answers with `RETRY`).

**Steps:**
1. Submit the question `"test-force-fail"` with no corpus tag.

**Expected:**
- Run progresses `ANSWERING` → `EVALUATING` → `ANSWERING` → `EVALUATING` → … for `maxAttempts` cycles (default 3).
- After the 3rd cycle ends in `RETRY`, the run transitions to `FAILED_FINAL` (not stuck in `EVALUATING`).
- The terminal block shows the highest-scoring attempt's answer as "best of 3 attempts" and the `failureReason` reads `"max attempts reached (3)"`.
- All 3 attempts are present in the expanded view, each with its answer, grounding `OK` verdict, `RETRY` score, and per-metric scores.
- `GET /api/runs/{id}` returns the full `EvalRun` with all 3 attempts in `attempts[]` and `status: "FAILED_FINAL"`.

## J3 — Grounding guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the question `"test-low-grounding"` (the mock provider's `seedFor` logic returns `groundingConfidence=0.20` for this question on attempt 1).

**Expected:**
- Attempt 1's answer has `groundingConfidence=0.20`, which is below the configured floor (0.40). The grounding step records `verdict.passed = false`, `reasonCode = "BELOW_CONFIDENCE_FLOOR"`, with detail naming the actual and floor values.
- The RAGAS Evaluator is NOT called for attempt 1. The run stays in `ANSWERING`.
- The Retrieval agent is called again with structured feedback (`"Answer grounding confidence below minimum floor; refine retrieval query and resubmit."`). Attempt 2 has higher confidence and passes the grounding check.
- The evaluator scores attempt 2 normally. The loop continues until `PASS` or the retry ceiling.
- The expanded view shows attempt 1 with the answer and a red `BELOW_CONFIDENCE_FLOOR` pill, attempt 2 with `OK`, and so on.

## J4 — Score-event timeline

**Preconditions:** At least one run has completed (any terminal state).

**Steps:**
1. Click the run card to expand.

**Expected:**
- The timeline shows one `ScoreRecorded` event per RAGAS-evaluated attempt, with `verdict`, `faithfulness`, `answerRelevance`, `contextPrecision`, and `ceilingExceeded` populated.
- The terminal transition (`RunPassed` or `RunFailedFinal`) is also surfaced as a final `ScoreRecorded` event carrying the loop-level outcome.
- `GET /api/runs/{id}` includes the equivalent data so the UI can render the timeline without a separate fetch.

## J5 — CI gate pass-rate check

**Preconditions:** At least 10 completed runs exist in `EvalRunsView`.

**Steps:**
1. Call `GET /api/ci/gate-status`.

**Expected:**
- Response body contains `{ "gatePassed": Boolean, "passRate": Double, "windowSize": Integer }`.
- `windowSize` is the number of completed runs considered (up to 100).
- If 80% or more of the runs in the window have `status: "PASSED"`, `gatePassed` is `true`; otherwise `false`.
- The CI gate widget on the App UI shows the same `passRate` as a percentage bar and a matching `Gate open` / `Gate blocked` badge.
- Injecting 25 additional mock `FAILED_FINAL` runs (such that the window pass rate falls below 0.80) causes a subsequent `GET /api/ci/gate-status` to return `"gatePassed": false`.
