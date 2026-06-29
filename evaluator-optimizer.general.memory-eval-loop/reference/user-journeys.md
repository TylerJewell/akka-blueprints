# User journeys — memory-eval-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9774; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9774/`. App UI tab is visible.
2. In the Question field, type "What programming languages do I use at work?". Leave User ID as "anonymous". Click Submit.
3. A new session turn card appears with status `ANSWERING`.

**Expected:**
- Within 1 s, the first attempt's answer appears in the expanded timeline.
- The cited entry IDs appear as chips; if memory entries exist for "anonymous", at least one entry is cited.
- Within 60 s of submission, either:
  - Attempt 1's verdict is `PASS` and the session turns to `ACCEPTED`, OR
  - Attempt 1's verdict is `IMPROVE` with 1–3 bullets and the session transitions back to `ANSWERING`, with attempt 2 appearing shortly after; this continues until either a `PASS` or the retry ceiling is reached.
- On `ACCEPTED`, the terminal block shows the accepted answer and "best of N attempts."
- The expanded view shows every attempt's answer text, cited entry IDs, scorer verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Scorer to always return `IMPROVE` (test mode — submit the literal question `"test-force-reject"`, which the mock provider's `seedFor` logic always answers with `IMPROVE`).

**Steps:**
1. Submit the question `"test-force-reject"` with the default User ID.

**Expected:**
- Session turn progresses `ANSWERING` → `SCORING` → `ANSWERING` → `SCORING` → … for `maxAttempts` cycles (default 3).
- After the 3rd cycle ends in `IMPROVE`, the session turns to `REJECTED_FINAL` (not stuck in `SCORING`).
- The terminal block shows the highest-scoring attempt's answer as the "best of 3 attempts" and `rejectionReason` reads `"max attempts reached (3)"`.
- All 3 attempts are present in the expanded view, each with its answer, `IMPROVE` verdict, and score.
- `GET /api/sessions/{id}` returns the full Session with all 3 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — PII sanitization

**Preconditions:** As J1.

**Steps:**
1. Submit the question `"remember that my email is alice@example.com"`. The mock or real agent will include this string in the memory write payload.

**Expected:**
- The session turn proceeds to `ACCEPTED` normally.
- `GET /api/memory/anonymous` does NOT include `alice@example.com` in any entry's `content` field.
- The entry's content instead contains `[REDACTED:EMAIL]` at the position of the email address.
- The `MemoryStore` event log contains a `MemoryEntryPiiRedacted` event for this session, confirming the sanitizer ran.
- The redacted entry IS present in `MemoryView` (the memory is stored, just sanitized), and the App UI's memory panel shows it with the redaction token visible.

## J4 — Drift check fires and surfaces in the UI

**Preconditions:** At least 5 session turns have completed (any terminal state). The `DriftWatcher` interval is 10 minutes; in test mode, trigger it manually by calling the internal endpoint `POST /api/sessions/trigger-drift-check` (test-only endpoint, not documented in the public contract).

**Steps:**
1. Ensure 5 or more session turns are in a terminal state.
2. Wait for the `DriftWatcher` to fire (10-minute interval), or trigger it via the test endpoint.
3. Observe the drift panel at the bottom of the App UI tab.

**Expected:**
- The drift panel shows a `DriftCheckRecorded` row with `passRate`, `avgScore`, and `driftFlagged` populated.
- If the pass rate across the 5 turns is at or above 0.60, `driftFlagged` is `false` and the panel renders in normal colours.
- If the pass rate is below 0.60 (achievable by submitting `"test-force-reject"` enough times), `driftFlagged` is `true` and the panel renders the pass-rate value in yellow.
- `GET /api/sessions/drift-watch-singleton` returns the sentinel session row with a non-empty `driftHistory` array.
