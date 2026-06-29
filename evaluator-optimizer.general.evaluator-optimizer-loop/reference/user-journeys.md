# User journeys — evaluator-optimizer-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9401; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9401/`. App UI tab is visible.
2. In the Problem statement field, type "Explain why exponential backoff is preferred over fixed-interval retry in distributed systems." Leave the token ceiling at the default 500. Click Submit.
3. A new job card appears with status `GENERATING`.

**Expected:**
- Within 1 s, status transitions to `GENERATING` (already there) and the first attempt's candidate appears.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's evaluation is `ACCEPT` and the job transitions to `ACCEPTED`, OR
  - Attempt 1's evaluation is `REVISE` with 1–3 bullets and the job transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either an `ACCEPT` or the attempt ceiling is reached.
- On `ACCEPTED`, the terminal block shows the accepted text and "best of N attempts."
- The expanded view shows every attempt's candidate, guardrail verdict, evaluator verdict, score, and notes.

## J2 — Halt at attempt ceiling

**Preconditions:** As J1, plus an override that forces the Evaluator to always return `REVISE` (test mode — submit the literal description `"test-force-reject"`, which the mock provider's `seedFor` logic always answers with `REVISE`).

**Steps:**
1. Submit the description `"test-force-reject"` with the default token ceiling.

**Expected:**
- Job progresses `GENERATING` → `EVALUATING` → `GENERATING` → `EVALUATING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the job transitions to `REJECTED_FINAL` (not stuck in `EVALUATING`).
- The terminal block shows the highest-scoring attempt's text as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its candidate, guardrail OK verdict, REVISE evaluation, and score.
- `GET /api/jobs/{id}` returns the full Job with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the problem `"Summarise the history of computing in one sentence"` with token ceiling **50**.

**Expected:**
- Attempt 1's candidate is over the ceiling. The guardrail records `verdict.passed = false`, `reasonCode = "OVER_CEILING"`, with detail naming the excess.
- The Evaluator is NOT called for attempt 1. The job stays in `GENERATING`.
- The Generator is called again with structured feedback (`"Candidate exceeds the configured token ceiling; shorten and resubmit."`). Attempt 2 is shorter and passes the guardrail.
- The evaluator scores attempt 2 normally. The loop continues until `ACCEPT` or the attempt ceiling.
- The expanded view shows attempt 1 with the over-ceiling text and the red `OVER_CEILING` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one job has completed (any terminal state).

**Steps:**
1. Click the job card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per evaluated attempt, with `verdict`, `score`, and `ceilingExceeded` populated.
- The terminal transition (JobAccepted or JobRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/jobs/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
