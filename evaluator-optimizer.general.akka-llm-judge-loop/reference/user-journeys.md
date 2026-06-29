# User journeys — llm-judge-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9358; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9358/`. App UI tab is visible.
2. In the Question field, type "What causes the aurora borealis?". Leave domain tag at `general` and score threshold at 4. Click Submit.
3. A new evaluation card appears with status `GENERATING`.

**Expected:**
- Within 1 s, status transitions to `GENERATING` (already there) and the first attempt's answer appears.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's judgment is `ACCEPT` and the evaluation transitions to `ACCEPTED`, OR
  - Attempt 1's judgment is `REVISE` with 1–3 bullets and the evaluation transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either an `ACCEPT` or the retry ceiling is reached.
- On `ACCEPTED`, the terminal block shows the accepted answer text and "best of N attempts."
- The expanded view shows every attempt's answer, guardrail verdict, judge verdict, score, and feedback.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Judge to always return `REVISE` (test mode — submit the literal question `"test-force-reject"`, which the mock provider's `seedFor` logic always answers with `REVISE`).

**Steps:**
1. Submit the question `"test-force-reject"` with the default score threshold.

**Expected:**
- Evaluation progresses `GENERATING` → `JUDGING` → `GENERATING` → `JUDGING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the evaluation transitions to `REJECTED_FINAL` (not stuck in `JUDGING`).
- The terminal block shows the highest-scoring attempt's answer as the "best of 4 attempts" and `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its answer, guardrail `OK` verdict, `REVISE` judgment, and score.
- `GET /api/evaluations/{id}` returns the full Evaluation with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the question `"test-guardrail-fail"` — the mock provider's `seedFor` logic selects the under-threshold answer (tokenCount < 50) for attempt 1 when this exact question text is used.

**Expected:**
- Attempt 1's answer has `tokenCount < 50`. The guardrail records `verdict.passed = false`, `reasonCode = "STRUCTURAL_FAIL"`, with detail naming the token count and minimum.
- The Judge is NOT called for attempt 1. The evaluation stays in `GENERATING`.
- The Generator is called again with structured feedback (`"Answer did not meet structural requirements; provide a complete response."`). Attempt 2's token count exceeds the minimum and passes the guardrail.
- The judge scores attempt 2 normally. The loop continues until `ACCEPT` or the retry ceiling.
- The expanded view shows attempt 1 with the low-token answer and the red `STRUCTURAL_FAIL` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one evaluation has completed (any terminal state).

**Steps:**
1. Click the evaluation card to expand.

**Expected:**
- The timeline shows one `JudgmentRecorded` event per judged attempt, with `verdict`, `score`, and `guardrailFailed` populated.
- The terminal transition (`EvaluationAccepted` or `EvaluationRejectedFinal`) is also surfaced as a final `JudgmentRecorded` event carrying the loop-level outcome.
- `GET /api/evaluations/{id}` includes a surfaced representation of the judgment events — either an `evalEvents[]` array or embedded in the SSE payload — with the same content. The UI does not require a separate fetch to render the timeline.
