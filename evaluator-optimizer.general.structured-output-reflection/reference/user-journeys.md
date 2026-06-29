# User journeys — structured-output-reflection

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9988; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9988/`. App UI tab is visible.
2. Select schema "product". Enter prompt "a professional-grade wireless keyboard for developers". Click Submit.
3. A new generation card appears with status `GENERATING`.

**Expected:**
- Within 1 s, status transitions to `GENERATING` (already there) and the first attempt's output appears.
- The schema guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's validation report is `PASS` and the generation transitions to `PASSED`, OR
  - Attempt 1's report is `REVISE` with 1–3 bullets and the generation transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either a `PASS` or the retry ceiling is reached.
- On `PASSED`, the terminal block shows the accepted JSON and "passed on attempt N."
- The expanded view shows every attempt's output, guardrail verdict, critic verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Critic to always return `REVISE` (test mode — submit the literal prompt `"test-force-fail"` with schema `product`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Select schema "product". Submit the prompt `"test-force-fail"`.

**Expected:**
- Generation progresses `GENERATING` → `VALIDATING` → `GENERATING` → `VALIDATING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the generation transitions to `FAILED_FINAL` (not stuck in `VALIDATING`).
- The terminal block shows the highest-scoring attempt's document as the "best of 4 attempts" and the `failureReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its output, guardrail OK verdict, REVISE validation report, and score.
- `GET /api/generations/{id}` returns the full Generation with all 4 attempts in `attempts[]` and `status: "FAILED_FINAL"`.

## J3 — Schema guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Select schema "product". Submit the prompt `"test-schema-invalid"` (the mock provider's seedFor logic returns an intentionally malformed JSON document on attempt 1 for this prompt).

**Expected:**
- Attempt 1's output has `parseable = false`. The guardrail records `verdict.passed = false`, `reasonCode = "SCHEMA_INVALID"`, with detail describing the structural failure.
- The Critic is NOT called for attempt 1. The generation stays in `GENERATING`.
- The Generator is called again with structured feedback (`"Output is not valid JSON or is missing required schema fields; correct and resubmit."`). Attempt 2 is well-formed and passes the guardrail.
- The critic scores attempt 2 normally. The loop continues until `PASS` or the retry ceiling.
- The expanded view shows attempt 1 with the malformed output and the red `SCHEMA_INVALID` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one generation has completed (any terminal state).

**Steps:**
1. Click the generation card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per validated attempt, with `verdict`, `score`, and `schemaFailed` populated.
- The terminal transition (GenerationPassed or GenerationFailedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/generations/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
