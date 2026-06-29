# User journeys — pr-reviewer

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9610; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9610/`. App UI tab is visible.
2. Paste a unified diff for a small Java change into the diff textarea. Leave description empty. Click Submit.
3. A new review card appears with status `REVIEWING`.

**Expected:**
- Within 1 s, status transitions to `REVIEWING` (already there) and the first attempt's feedback comments appear.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's alignment check is `APPROVE` and the review transitions to `APPROVED`, OR
  - Attempt 1's alignment check is `REVISE` with 1–3 bullets and the review transitions back to `REVIEWING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved comments and "best of N attempts."
- The expanded view shows every attempt's feedback, guardrail verdict, alignment verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Alignment agent to always return `REVISE` (test mode — submit the description `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit a diff with description `"test-force-reject"`.

**Expected:**
- Review progresses `REVIEWING` → `CHECKING_ALIGNMENT` → `REVIEWING` → `CHECKING_ALIGNMENT` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the review transitions to `REJECTED_FINAL` (not stuck in `CHECKING_ALIGNMENT`).
- The terminal block shows the highest-scoring attempt's feedback as "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its feedback, guardrail OK verdict, REVISE alignment check, and score.
- `GET /api/reviews/{id}` returns the full Review with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit a diff where the mock Reviewer's first draft contains a personal critique comment (e.g., a comment with "you should know better than this"). This is deterministically triggered in mock mode by submitting a diff description of `"test-personal-critique"`.

**Expected:**
- Attempt 1's feedback contains a comment with personal language. The guardrail records `verdict.passed = false`, `reasonCode = "PERSONAL_CRITIQUE"`, with detail naming the offending comment.
- The Alignment agent is NOT called for attempt 1. The review stays in `REVIEWING`.
- The Reviewer is called again with structured feedback ("Feedback contains personal critique directed at the author; rewrite as impersonal observations about the code."). Attempt 2's comments are impersonal.
- The alignment agent evaluates attempt 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows attempt 1 with the personal-critique feedback and the red `PERSONAL_CRITIQUE` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one review has completed (any terminal state).

**Steps:**
1. Click the review card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per alignment-checked attempt, with `verdict`, `score`, and `personalCritiqueBlocked` populated.
- The terminal transition (ReviewApproved or ReviewRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/reviews/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — Simulator populates the list without user action

**Preconditions:** Service running on port 9610 with no prior user submissions.

**Steps:**
1. Open `http://localhost:9610/`. App UI tab is visible.
2. Wait up to 65 seconds without submitting anything.

**Expected:**
- At least one review card appears automatically (from `PrSimulator`).
- The card goes through the normal `REVIEWING` → `CHECKING_ALIGNMENT` → terminal state lifecycle, observable via SSE.
- The review card can be expanded to show the per-attempt timeline as in J1.
