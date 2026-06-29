# User journeys — on-brand-genmedia

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9536; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9536/`. App UI tab is visible.
2. In the Product field, type "Nova Project Management Platform". Set Channel to "LinkedIn", Tone to "professional". Leave the token ceiling at the default 300. Click Submit.
3. A new asset card appears with status `GENERATING`.

**Expected:**
- Within 1 s, the first attempt's generated content appears (headline, body copy, social caption).
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's review is `APPROVE` and the asset transitions to `APPROVED`, OR
  - Attempt 1's review is `REVISE` with 1–3 bullets and the asset transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved headline, body copy, and social caption, plus a "best of N attempts" caption.
- The expanded view shows every attempt's generated content, guardrail verdict, reviewer verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the reviewer to always return `REVISE` (test mode — submit the literal product `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit with product `"test-force-reject"`, channel `"LinkedIn"`, tone `"professional"`.

**Expected:**
- Asset progresses `GENERATING` → `REVIEWING` → `GENERATING` → `REVIEWING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the asset transitions to `REJECTED_FINAL` (not stuck in `REVIEWING`).
- The terminal block shows the highest-scoring attempt's asset as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its generated content, guardrail OK verdict, REVISE review, and score.
- `GET /api/assets/{id}` returns the full Asset with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail block on prohibited content

**Preconditions:** As J1.

**Steps:**
1. Submit with product `"test-prohibited-content"` — the mock provider's seedFor logic returns an asset containing "world's best" in the headline for attempt 1.

**Expected:**
- Attempt 1's asset contains a prohibited term. The guardrail records `verdict.passed = false`, `reasonCode = "PROHIBITED_CONTENT"`, with detail naming the offending term.
- The reviewer is NOT called for attempt 1. The asset stays in `GENERATING`.
- The brand agent is called again with structured feedback (`"Generated content contains prohibited terms; revise to remove them."`). Attempt 2 is clean and passes the guardrail.
- The reviewer scores attempt 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows attempt 1 with the prohibited-content content and the red `PROHIBITED_CONTENT` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one asset has completed (any terminal state).

**Steps:**
1. Click the asset card to expand.

**Expected:**
- The timeline shows one `ReviewEvalRecorded` event per reviewed attempt, with `verdict`, `score`, and `guardrailFailed` populated.
- The terminal transition (AssetApproved or AssetRejectedFinal) is also surfaced as a final `ReviewEvalRecorded` event carrying the loop-level outcome.
- `GET /api/assets/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
