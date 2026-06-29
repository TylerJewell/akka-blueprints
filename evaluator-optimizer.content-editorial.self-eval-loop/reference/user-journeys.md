# User journeys — self-eval-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9346; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9346/`. App UI tab is visible.
2. In the Brief field, type "the first frost of October on a city park". Leave the character ceiling at the default 280. Click Submit.
3. A new post card appears with status `DRAFTING`.

**Expected:**
- Within 1 s, status transitions to `DRAFTING` (already there) and the first attempt's draft appears.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's critique is `ACCEPT` and the post transitions to `ACCEPTED`, OR
  - Attempt 1's critique is `REVISE` with 1–3 bullets and the post transitions back to `DRAFTING`, with attempt 2 appearing shortly after; this continues until either an `ACCEPT` or the retry ceiling is reached.
- On `ACCEPTED`, the terminal block shows the accepted text and "best of N attempts."
- The expanded view shows every attempt's draft, guardrail verdict, critic verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Critic to always return `REVISE` (test mode — submit the literal brief `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the brief `"test-force-reject"` with the default ceiling.

**Expected:**
- Post progresses `DRAFTING` → `EVALUATING` → `DRAFTING` → `EVALUATING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the post transitions to `REJECTED_FINAL` (not stuck in `EVALUATING`).
- The terminal block shows the highest-scoring attempt's text as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its draft, guardrail OK verdict, REVISE critique, and score.
- `GET /api/posts/{id}` returns the full Post with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the brief `"the longest possible bombastic dedication"` with character ceiling **80**.

**Expected:**
- Attempt 1's draft is over the ceiling. The guardrail records
  `verdict.passed = false`, `reasonCode = "OVER_CEILING"`, with detail naming the excess.
- The Critic is NOT called for attempt 1. The post stays in `DRAFTING`.
- The Poet is called again with structured feedback (`"Draft exceeds the configured character ceiling; trim and resubmit."`). Attempt 2 is shorter and passes the guardrail.
- The critic scores attempt 2 normally. The loop continues until `ACCEPT` or the retry ceiling.
- The expanded view shows attempt 1 with the over-ceiling text and the red `OVER_CEILING` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one post has completed (any terminal state).

**Steps:**
1. Click the post card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per critiqued attempt, with `verdict`, `score`, and `ceilingExceeded` populated.
- The terminal transition (PostAccepted or PostRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/posts/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
