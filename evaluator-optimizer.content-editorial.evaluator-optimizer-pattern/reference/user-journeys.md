# User journeys — evaluator-optimizer-workflow

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9404; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9404/`. App UI tab is visible.
2. In the summary field, type "Researchers found that daily 15-minute walks significantly reduce cardiovascular disease risk in adults over sixty." Leave the word ceiling at the default 12. Click Submit.
3. A new headline card appears with status `DRAFTING`.

**Expected:**
- Within 1 s, status transitions to `DRAFTING` (already there) and the first attempt's draft appears.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's review is `APPROVE` and the headline transitions to `APPROVED`, OR
  - Attempt 1's review is `REVISE` with 1–3 bullets and the headline transitions back to `DRAFTING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved headline text and "best of N attempts."
- The expanded view shows every attempt's draft, guardrail verdict, reviewer verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Reviewer to always return `REVISE` (test mode — submit the literal summary `"test-force-reject"`, which the mock provider's `seedFor` logic always answers with `REVISE`).

**Steps:**
1. Submit the summary `"test-force-reject"` with the default word ceiling.

**Expected:**
- Headline progresses `DRAFTING` → `REVIEWING` → `DRAFTING` → `REVIEWING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the headline transitions to `REJECTED_FINAL` (not stuck in `REVIEWING`).
- The terminal block shows the highest-scoring attempt's headline text as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its draft, guardrail OK verdict, REVISE review, and score.
- `GET /api/headlines/{id}` returns the full Headline with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the summary `"Scientists have discovered a groundbreaking treatment"` with word ceiling **5**.

**Expected:**
- Attempt 1's draft exceeds 5 words. The guardrail records `verdict.passed = false`, `reasonCode = "OVER_CEILING"`, with detail naming the excess word count.
- The Reviewer is NOT called for attempt 1. The headline stays in `DRAFTING`.
- The Writer is called again with structured feedback (`"Draft exceeds the configured word ceiling; shorten and resubmit."`). Attempt 2 is within the ceiling and passes the guardrail.
- The reviewer scores attempt 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows attempt 1 with the over-ceiling text and the red `OVER_CEILING` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one headline has completed (any terminal state).

**Steps:**
1. Click the headline card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per reviewed attempt, with `verdict`, `score`, and `ceilingExceeded` populated.
- The terminal transition (HeadlineApproved or HeadlineRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/headlines/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
