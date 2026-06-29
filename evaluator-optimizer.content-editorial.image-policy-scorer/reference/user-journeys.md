# User journeys — image-policy-scorer

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9987; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9987/`. App UI tab is visible.
2. In the Prompt field, type "a street food market at dusk in a busy city". Leave the audience tier at `general`. Click Submit.
3. A new image card appears with status `GENERATING`.

**Expected:**
- Within 1 s the first attempt's description appears in the expanded view.
- The safety gate verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's policy verdict is `PASS` and the image transitions to `APPROVED`, OR
  - Attempt 1's policy verdict is `FAIL` with 1–3 bullets and the image transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either a `PASS` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved description and "best of N attempts."
- The expanded view shows every attempt's description, gate verdict, scorer verdict, score, and policy notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Scorer to always return `FAIL` (test mode — submit the literal prompt `"test-force-reject"`, which the mock provider's seedFor logic always answers with `FAIL`).

**Steps:**
1. Submit the prompt `"test-force-reject"` with audience tier `general`.

**Expected:**
- Image progresses `GENERATING` → `SCORING` → `GENERATING` → `SCORING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `FAIL`, the image transitions to `REJECTED_FINAL` (not stuck in `SCORING`).
- The terminal block shows the highest-scoring attempt's description as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its description, gate OK verdict, FAIL policy verdict, and score.
- `GET /api/images/{id}` returns the full Image with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Safety gate short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit a prompt that the mock provider's prohibited entry answers with `brandSafetySignal = "violence"` (use the literal prompt `"test-force-prohibited"` in mock mode, or any prompt that elicits a `violence` signal in live mode).

**Expected:**
- Attempt 1's description has `brandSafetySignal = "violence"`. The safety gate records
  `verdict.passed = false`, `reasonCode = "PROHIBITED_SIGNAL"`, with detail naming the matched signal.
- The Scorer is NOT called for attempt 1. The image stays in `GENERATING`.
- The Generator is called again with structured feedback (`"Description contains a prohibited brand-safety signal; revise and resubmit."`). Attempt 2 produces a description with `brandSafetySignal = "none"` or `"mild"` that passes the gate.
- The scorer evaluates attempt 2 normally. The loop continues until `PASS` or the retry ceiling.
- The expanded view shows attempt 1 with the `PROHIBITED_SIGNAL` red pill and no scorer verdict, attempt 2 with `OK` and its scorer verdict, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one image has completed (any terminal state).

**Steps:**
1. Click the image card to expand.

**Expected:**
- The timeline shows one `PolicyEvalRecorded` event per scored attempt, with `decision`, `score`, and `gateBlocked` populated.
- The terminal transition (ImageApproved or ImageRejectedFinal) is also surfaced as a final `PolicyEvalRecorded` event carrying the loop-level outcome.
- `GET /api/images/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — Audience-tier enforcement

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "a vibrant nightlife scene with cocktail bars" with audience tier `children`.

**Expected:**
- The Scorer's rubric applies `children` tier restrictions. The audience-tier dimension of the rubric scores 1 for any description that includes nightlife or bar imagery.
- Attempt 1 receives `FAIL` with a bullet citing the audience-tier mismatch.
- The Generator revises to a daytime, family-appropriate scene.
- The loop converges on `APPROVED` or reaches `REJECTED_FINAL` after `maxAttempts` cycles.
- In both cases, the terminal block correctly reflects the final state and every attempt's policy notes are preserved.
