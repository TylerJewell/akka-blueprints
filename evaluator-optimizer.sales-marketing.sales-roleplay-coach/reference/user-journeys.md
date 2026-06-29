# User journeys — sales-roleplay-coach

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9300; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9300/`. App UI tab is visible.
2. Enter buyer persona "VP of Engineering at a mid-market SaaS company", product "AI-powered observability platform", deal stage DEMO. Click Start session.
3. A new session card appears with status `PITCHING` and the buyer's opening statement.
4. In the pitch input area, type a pitch turn addressing the buyer's opening. Click Submit turn.

**Expected:**
- Within 1 s, status transitions to `EVALUATING` and the first turn's guardrail verdict appears as `OK`.
- The buyer's response appears with a signal badge.
- The coach's verdict for turn 1 appears: either `ACCEPT` (session moves to `PASSED`) or `REVISE` (session returns to `PITCHING` with coaching bullets visible).
- If `REVISE`, the pitch input area re-appears for turn 2. Submitting an improved turn restarts the cycle.
- On `PASSED`, the terminal block shows the accepted pitch text and a "passed on turn N of maxTurns" caption.
- The expanded view shows every turn's rep text, guardrail verdict, buyer response, coach decision, score, and notes.

## J2 — Halt at turn ceiling

**Preconditions:** As J1, plus test mode that forces the coach to always return `REVISE` (submit buyer persona `"test-force-exhaust"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Start a session with buyer persona `"test-force-exhaust"`, any product, deal stage DEMO.
2. Submit a pitch turn for each prompted turn until the session ends.

**Expected:**
- Session progresses `PITCHING` → `EVALUATING` → `PITCHING` → `EVALUATING` → … for `maxTurns` cycles (default 5).
- After the 5th turn ends in `REVISE`, the session transitions to `EXHAUSTED` (not stuck in `EVALUATING`).
- The terminal block shows the highest-scoring turn's text as the "best of 5 turns" and the `exhaustionReason` reads `"max turns reached (5)"`.
- All 5 turns are present in the expanded view, each with its rep text, guardrail OK verdict, buyer response, REVISE coaching verdict, and score.
- `GET /api/sessions/{id}` returns the full Session with all 5 turns in `turns[]` and `status: "EXHAUSTED"`.

## J3 — Guardrail block on prohibited content

**Preconditions:** As J1.

**Steps:**
1. Start a session with buyer persona "CFO at a regional bank", product "AI fraud-detection platform", deal stage PROPOSAL.
2. When the pitch input area appears, submit a turn that contains a competitor brand claim (e.g., "Unlike [CompetitorName], our system...").

**Expected:**
- The guardrail records `verdict.passed = false`, `reasonCode = "PROHIBITED_CONTENT"`, with `detail` identifying the prohibited pattern.
- The buyer simulator and coach are NOT called for this turn. The session stays in `PITCHING`.
- The pitch input area re-appears with the feedback note: "Turn contains prohibited content; remove competitor brand claims and unverified pricing before resubmitting."
- The turn still counts toward `maxTurns`.
- The rep submits a revised turn without the prohibited content. The guardrail passes, the loop continues normally.
- The expanded view shows the blocked turn with the red `PROHIBITED_CONTENT` pill, the revised turn with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one session has reached `PASSED` or `EXHAUSTED`.

**Steps:**
1. Click any completed session card to expand it.

**Expected:**
- The timeline shows one `EvalRecorded` event per coached turn (i.e., turns that passed the guardrail and received a coaching verdict), with `decision`, `score`, and `guardrailFlagged` populated.
- The terminal transition (SessionPassed or SessionExhausted) is also surfaced as a final `EvalRecorded` event carrying the session-level outcome.
- `GET /api/sessions/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — ScenarioSimulator populates the list

**Preconditions:** Service running; no sessions created manually.

**Steps:**
1. Open `http://localhost:9300/` and switch to the App UI tab.
2. Wait up to 90 seconds without submitting anything.

**Expected:**
- At least one session card appears, created by the `ScenarioSimulator` TimedAction from `sales-scenarios.jsonl`.
- The card's buyer persona and product match one of the 8 canned scenarios in the JSONL file.
- The session is in `PITCHING` (the buyer's opening statement is visible) or has progressed further if the mock mode responded automatically.

## J6 — Rep turn rejected after session is terminal

**Preconditions:** A session in `PASSED` or `EXHAUSTED` state exists.

**Steps:**
1. Issue `POST /api/sessions/{id}/turns` with any text body against the terminal session's ID.

**Expected:**
- The API returns `409 Conflict`.
- The session's state does not change.
- No new event is emitted on `SessionEntity`.
