# User journeys — brand-presentation-builder

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9366; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9366/`. App UI tab is visible.
2. In the Topic field, type "launch of the new AI-assisted analytics dashboard". Set Target audience to "enterprise sales team". Leave Slides at 5 and Words per slide at 80. Click Submit.
3. A new presentation card appears with status `BUILDING`.

**Expected:**
- Within 1 s, status transitions to `BUILDING` (already there) and the first attempt's slide set appears as a collapsed accordion.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 90 s of submission, either:
  - Attempt 1's review is `APPROVE` and the presentation transitions to `APPROVED`, OR
  - Attempt 1's review is `REVISE` with 1–5 bullets and the presentation transitions back to `BUILDING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved slide set and "best of N attempts."
- The expanded view shows every attempt's slide set, guardrail verdict, reviewer verdict, score, and brand feedback.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Reviewer to always return `REVISE` (test mode — submit the literal topic `"test-force-reject"`, which the mock provider's `seedFor` logic always answers with `REVISE`).

**Steps:**
1. Submit the topic `"test-force-reject"` with the default audience, slide count, and word ceiling.

**Expected:**
- Presentation progresses `BUILDING` → `REVIEWING` → `BUILDING` → `REVIEWING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the presentation transitions to `REJECTED_FINAL` (not stuck in `REVIEWING`).
- The terminal block shows the highest-scoring attempt's slide set as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its slide set, guardrail `OK` verdict, `REVISE` brand review, and score.
- `GET /api/presentations/{id}` returns the full Presentation with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the topic `"comprehensive enterprise AI transformation strategy"` with words per slide set to **30**.

**Expected:**
- Attempt 1's slide set includes at least one slide whose word count exceeds 30. The guardrail records `verdict.passed = false`, `reasonCode = "OVER_WORD_LIMIT"`, with detail naming the offending slide numbers.
- The Brand Reviewer is NOT called for attempt 1. The presentation stays in `BUILDING`.
- The Builder is called again with structured feedback naming the over-limit slides. Attempt 2 is revised to stay within 30 words per slide.
- The reviewer scores attempt 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows attempt 1 with the slide set and the red `OVER_WORD_LIMIT` pill naming offending slides, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one presentation has completed (any terminal state).

**Steps:**
1. Click the presentation card to expand.

**Expected:**
- The timeline shows one `BrandEvalRecorded` event per reviewed attempt, with `verdict`, `score`, and `guardedSlideNumbers` populated.
- The terminal transition (PresentationApproved or PresentationRejectedFinal) is also surfaced as a final `BrandEvalRecorded` event carrying the loop-level outcome.
- `GET /api/presentations/{id}` includes the `BrandEvalRecorded` events accessible through the SSE update, with the same content. The UI does not require a separate fetch to render the timeline.
