# User journeys — competitive-coding-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9346; valid model-provider API key set and `E2B_API_KEY` set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9346/`. App UI tab is visible.
2. In the Title field, type "Prefix Sum Maximum". In the Problem statement textarea, paste a short USACO-style problem with two sample test cases. Leave time limit and memory limit at defaults. Click Submit.
3. A new submission card appears with status `GENERATING`.

**Expected:**
- Within 2 s, status transitions to `GENERATING` (already there) and the first attempt's solution appears.
- The sandbox verdict pill on attempt 1 reads `OK` with a wall time and peak memory badge.
- Within 120 s of submission, either:
  - Attempt 1's judge verdict is `PASS` and the submission transitions to `ACCEPTED`, OR
  - Attempt 1's verdict is `FAIL` with 1–3 bullets and the submission transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either a `PASS` or the retry ceiling is reached.
- On `ACCEPTED`, the terminal block shows the accepted source code and "passed on attempt N."
- The expanded view shows every attempt's solution, sandbox report, judge verdict, case pass count, and failure notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Judge to always return `FAIL` (test mode — submit the literal title `"test-force-reject"`, which the mock provider's seedFor logic always answers with `FAIL` for every attempt).

**Steps:**
1. Submit a problem with title `"test-force-reject"` and minimal sample cases.

**Expected:**
- Submission progresses `GENERATING` → `JUDGING` → `GENERATING` → `JUDGING` → … for `maxAttempts` cycles (default 5).
- After the 5th cycle ends in `FAIL`, the submission transitions to `REJECTED_FINAL` (not stuck in `JUDGING`).
- The terminal block shows the highest case-pass-ratio attempt's source as "best of 5 attempts" and the `rejectionReason` reads `"max attempts reached (5)"`.
- All 5 attempts are present in the expanded view, each with its solution, sandbox OK, FAIL verdict, and failure notes.
- `GET /api/submissions/{id}` returns the full Submission with all 5 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Sandbox resource block

**Preconditions:** As J1.

**Steps:**
1. Submit a problem where the mock (or real) solver generates a solution with an infinite loop (mock: select the TLE entry from solver.json).

**Expected:**
- Attempt 1's sandbox report has `resourcesOk = false`, `reasonCode = "TLE"`, with detail naming the wall time exceeded.
- The JudgeAgent is NOT called for attempt 1. The submission stays in `GENERATING`.
- The SolverAgent is called again (REVISE_SOLUTION) with the structured feedback `"Solution exceeded resource limits; see reasonCode and detail."` Attempt 2 is generated with a more efficient algorithm.
- The sandbox report for attempt 2 has `resourcesOk = true`. The judge scores attempt 2 normally. The loop continues until `PASS` or the retry ceiling.
- The expanded view shows attempt 1 with the TLE pill and no judge verdict, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one submission has completed (any terminal state).

**Steps:**
1. Click the submission card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per judged attempt, with `outcome`, `passedCases`, `totalCases`, and `sandboxOk` populated.
- The terminal transition (SubmissionAccepted or SubmissionRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/submissions/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — Duplicate submission deduplication

**Preconditions:** Service running; at least one submission in any state.

**Steps:**
1. Submit the same problem title and `submittedBy` value twice within a 10-second window.

**Expected:**
- The second `POST /api/submissions` returns `200` (not `202`) with the same `submissionId` as the first call.
- Only one workflow is started. The submission list shows a single card for that problem.

## J6 — SSE live update

**Preconditions:** Service running; App UI open in browser.

**Steps:**
1. Keep the App UI open and submit a new problem.
2. Do not refresh the page.

**Expected:**
- As the submission progresses through states (`GENERATING` → `JUDGING` → `ACCEPTED`), the card updates in place without a page reload.
- Each attempt's content appears as soon as the corresponding SSE event is received.
- The status pill and attempt counter update live.
