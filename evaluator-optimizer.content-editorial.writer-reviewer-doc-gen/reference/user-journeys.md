# User journeys — writer-reviewer-doc-gen

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9885; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9885/`. App UI tab is visible.
2. In the Topic field, type "the role of open-source software in cloud infrastructure". Leave the word ceiling at the default 500. Click Submit.
3. A new document card appears with status `DRAFTING`.

**Expected:**
- Within 1 s, status transitions to `DRAFTING` (already there) and the first attempt's draft appears.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's review is `APPROVE` and the document transitions to `APPROVED`, OR
  - Attempt 1's review is `REVISE` with 1–3 bullets and the document transitions back to `DRAFTING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved text and "best of N attempts."
- The expanded view shows every attempt's draft, guardrail verdict, reviewer verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Reviewer to always return `REVISE` (test mode — submit the literal topic `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the topic `"test-force-reject"` with the default ceiling.

**Expected:**
- Document progresses `DRAFTING` → `REVIEWING` → `DRAFTING` → `REVIEWING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the document transitions to `REJECTED_FINAL` (not stuck in `REVIEWING`).
- The terminal block shows the highest-scoring draft's text as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its draft, guardrail OK verdict, REVISE review, and score.
- `GET /api/documents/{id}` returns the full Document with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the topic `"the complete history of distributed computing from 1960 to today"` with word ceiling **50**.

**Expected:**
- Attempt 1's draft is over the ceiling. The guardrail records `verdict.passed = false`, `reasonCode = "OVER_CEILING"`, with detail naming the excess word count.
- The Reviewer is NOT called for attempt 1. The document stays in `DRAFTING`.
- The Writer is called again with structured feedback (`"Draft exceeds the configured word ceiling; shorten and resubmit."`). Attempt 2 is shorter and passes the guardrail.
- The reviewer evaluates attempt 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows attempt 1 with the over-limit text and the red `OVER_CEILING` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one document has completed (any terminal state).

**Steps:**
1. Click the document card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per reviewed attempt, with `verdict`, `score`, and `ceilingExceeded` populated.
- The terminal transition (DocumentApproved or DocumentRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/documents/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
