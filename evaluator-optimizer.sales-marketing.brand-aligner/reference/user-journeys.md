# User journeys — brand-aligner

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9276; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9276/`. App UI tab is visible.
2. In the Topic field, type "Launch announcement for a new developer API that cuts integration time". Set Target Audience to "backend engineers". Leave the word ceiling at the default 150. Click Submit.
3. A new material card appears with status `DRAFTING`.

**Expected:**
- Within 1 s, status transitions to `DRAFTING` (already there) and the first attempt's variant appears.
- The compliance verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's review is `APPROVE` and the material transitions to `APPROVED`, OR
  - Attempt 1's review is `REVISE` with 1–3 bullets and the material transitions back to `DRAFTING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved text and "best of N attempts."
- The expanded view shows every attempt's variant, compliance verdict, reviewer verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Reviewer to always return `REVISE` (test mode — submit the literal brief `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the brief `"test-force-reject"` with audience "general" and the default ceiling.

**Expected:**
- Material progresses `DRAFTING` → `REVIEWING` → `DRAFTING` → `REVIEWING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the material transitions to `REJECTED_FINAL` (not stuck in `REVIEWING`).
- The terminal block shows the highest-scoring variant's text as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its variant, compliance `OK` verdict, `REVISE` review, and score.
- `GET /api/materials/{id}` returns the full Material with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Compliance check short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the brief `"product overview for a broad audience"` with word ceiling **20**.

**Expected:**
- Attempt 1's variant exceeds the ceiling. The compliance check records `verdict.passed = false`, `reasonCode = "OVER_WORD_CEILING"`, with detail naming the excess word count.
- The Reviewer is NOT called for attempt 1. The material stays in `DRAFTING`.
- The Copywriter is called again with structured feedback (`"Variant exceeds the configured word ceiling; shorten and resubmit."`). Attempt 2 is shorter and passes the compliance check.
- The reviewer scores attempt 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows attempt 1 with the over-ceiling text and the red `OVER_WORD_CEILING` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one material has completed (any terminal state).

**Steps:**
1. Click the material card to expand.

**Expected:**
- The timeline shows one `BrandEvalRecorded` event per reviewed attempt, with `verdict`, `score`, and `wordCeilingExceeded` populated.
- The terminal transition (MaterialApproved or MaterialRejectedFinal) is also surfaced as a final `BrandEvalRecorded` event carrying the loop-level outcome.
- `GET /api/materials/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
