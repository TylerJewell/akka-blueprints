# User journeys — story-refiner

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9223; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9223/`. App UI tab is visible.
2. In the Feature description field, type "Allow platform engineers to rotate service account keys without downtime." Set Target team to "platform". Click Submit.
3. A new story card appears with status `DRAFTING`.

**Expected:**
- Within 1 s, the card transitions to `DRAFTING` (already there) and the first attempt's draft fields appear.
- Within 60 s of submission, either:
  - Attempt 1's review is `APPROVE` and the story transitions to `APPROVED`, OR
  - Attempt 1's review is `REVISE` with 1–4 bullets and the story transitions back to `DRAFTING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling.
- On `APPROVED`, the terminal block shows the approved draft fields and a "best of N attempts" caption.
- The expanded view shows every attempt's role, goal, benefit, acceptance criteria, reviewer verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Reviewer to always return `REVISE` (test mode — submit the literal description `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit feature description `"test-force-reject"` with target team "platform".

**Expected:**
- Story progresses `DRAFTING` → `REVIEWING` → `DRAFTING` → `REVIEWING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the story transitions to `REJECTED_FINAL` (not stuck in `REVIEWING`).
- The terminal block shows the highest-scoring attempt's draft as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its draft fields and `REVISE` review.
- `GET /api/stories/{id}` returns the full Story with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Reviewer note incorporation

**Preconditions:** As J1.

**Steps:**
1. Submit feature description "Add a global search bar to the admin console so that admins can find users and settings in one place."
2. Observe that attempt 1's reviewer returns `REVISE` with a bullet identifying a compound goal (conflates user search and settings search).

**Expected:**
- The Drafter's attempt 2 addresses the bullet: `goal` is rewritten to focus on a single search scope (e.g., user search only), or the benefit is updated to justify the combined scope clearly.
- The reviewer's score on the `goal specificity` dimension improves from attempt 1 to attempt 2 (visible in the score chips on each attempt block).
- If attempt 2 is approved, the approved draft's goal field no longer conflates two intents. If another `REVISE` is returned, the bullet in attempt 2's review is different from attempt 1's, confirming the prior note was addressed.

## J4 — Eval-event timeline

**Preconditions:** At least one story has completed (any terminal state).

**Steps:**
1. Click the story card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per reviewed attempt, with `verdict`, `score`, and `fieldScores` populated.
- The terminal transition (StoryApproved or StoryRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/stories/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
