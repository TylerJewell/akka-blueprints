# User journeys — seo-rubric-critic

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the round ceiling

**Preconditions:** Service running on port 9638; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9638/`. App UI tab is visible.
2. Fill in the Title field with "How to Write an SEO-Optimised Blog Post in 2026", paste a 900-word article body, enter the target keyword "SEO blog post", and leave the word-count ceiling at the default 1500. Click Submit.
3. A new article card appears with status `SCORING`.

**Expected:**
- Within 1 s, the first round's status is visible in the expanded card.
- The guardrail verdict pill on round 1 reads `OK`.
- Within 120 s of submission, either:
  - Round 1's rubric score has `passed = true` and the article transitions to `APPROVED`, OR
  - Round 1 fails with a composite score and failing dimensions, the article transitions to `REVISING`, and round 2 appears shortly after; this continues until either `APPROVED` or the round ceiling is reached.
- On `APPROVED`, the terminal block shows the approved body text, "passed at round N", and the final composite score.
- The expanded view shows every round's draft, guardrail verdict, rubric score, failing dimensions, and improvement summary.

## J2 — Halt at round ceiling

**Preconditions:** As J1, plus an override that forces the RubricAgent to always return `passed = false` (test mode — submit the literal title `"test-force-fail"`, which the mock provider's seedFor logic always answers with `compositeScore = 35`).

**Steps:**
1. Submit an article with title `"test-force-fail"` and any non-empty body.

**Expected:**
- Article progresses `SCORING` → `REVISING` → `SCORING` → `REVISING` → … for `maxRounds` cycles (default 4).
- After the 4th scoring round ends with `passed = false`, the article transitions to `FAILED_FINAL` (not stuck in `REVISING`).
- The terminal block shows the highest composite-score draft's text and the `rejectionReason` reads `"max rounds reached (4); best score 35/100"`.
- All 4 rounds are present in the expanded view, each with its draft, guardrail `OK` verdict, `FAIL` rubric verdict, and score.
- `GET /api/articles/{id}` returns the full Article with all 4 rounds in `rounds[]` and `status: "FAILED_FINAL"`.

## J3 — Guardrail short-circuit on over-ceiling revision

**Preconditions:** As J1.

**Steps:**
1. Submit an article with word-count ceiling **500** and a body of approximately 700 words.

**Expected:**
- Round 1's draft is over the 500-word ceiling. The guardrail records `verdict.passed = false`, `reasonCode = "OVER_WORD_CEILING"`, with detail naming the excess word count.
- The RubricAgent is NOT called for round 1. The article remains in `SCORING` (or `REVISING` depending on workflow step naming) without advancing.
- The OptimizerAgent is called again with the structured feedback (`"Draft exceeds the configured word-count ceiling; shorten and resubmit."`). Round 2's draft is shorter and passes the guardrail.
- The rubric agent scores round 2 normally. The loop continues until `APPROVED` or the round ceiling.
- The expanded view shows round 1 with the over-ceiling draft and the red `OVER_WORD_CEILING` pill, round 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one article has completed (any terminal state).

**Steps:**
1. Click the article card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per scored round, with `verdict` (PASS/FAIL), `compositeScore`, and `wordCeilingExceeded` populated.
- The terminal transition (ArticleApproved or ArticleFailedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome (total rounds, final composite score, whether the halt fired).
- `GET /api/articles/{id}` includes the eval events either as a top-level `evalEvents[]` array or embedded in the round objects. The UI does not require a separate fetch to render the timeline.
