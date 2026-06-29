# User journeys

The acceptance bar. The generated system is correct when these pass. Journeys J1–J4 are the inline acceptance set named in SPEC.md Section 10; J5 and J6 cover the simulator and the operator halt.

## J1 — Start a session and see the first turn

**Preconditions:** the service is running on `http://localhost:9693`; a model provider is wired (or the mock provider is active).

**Steps:**
1. Open the App UI tab.
2. Enter topic `"The role of open-source models in enterprise AI adoption"`. Click Start.

**Expected:**
- The response carries a `sessionId`.
- Within a few seconds the session appears in the list in `CHATTING` status with `currentTurn` 1.
- The turn history holds at least the Researcher's turn-1 `TurnLine`, with a non-empty `message` and `flagged = false`.
- `GET /api/sessions/{id}` returns the same session.

## J2 — Consensus conclusion

**Preconditions:** topic is well-scoped and the mock provider (or live model) is configured to converge.

**Steps:**
1. Start a session with topic `"Benefits of event-sourcing for audit trails"`.
2. Watch the SSE-fed list.

**Expected:**
- Across successive turns the Researcher and Critic messages converge; the Critic acknowledges agreement.
- The session reaches `CONCLUDED` with `terminationReason = CONSENSUS` at or before turn twenty.
- `conclusionSummary` is non-empty and reflects the agreed points.
- No `TurnLine` ever has `flagged = true` (the guardrail held; no turn was rejected as harmful).

## J3 — Max-turns conclusion when no consensus is reached

**Preconditions:** topic is deliberately irresolvable (the mock provider or model keeps returning `CONTINUE` through turn twenty).

**Steps:**
1. Start a session with topic `"Is consensus achievable on every topic"`.
2. Watch the list.

**Expected:**
- The session reaches `CONCLUDED` with `terminationReason = MAX_TURNS_REACHED` at turn twenty.
- `conclusionSummary` is non-empty (a partial summary of the discussion state), even though full consensus was not reached.

## J4 — Every concluded session is scored

**Preconditions:** at least one session from J2 or J3 has reached `CONCLUDED`.

**Steps:**
1. Inspect a concluded session in the list (or `GET /api/sessions/{id}`).

**Expected:**
- Within a few seconds of concluding, `qualityScore` is non-null and `qualityNotes` is non-empty.
- For a `CONSENSUS` session, the score reflects topic coherence and the number of turns used.
- For a `MAX_TURNS_REACHED` session, the score reflects that no shared conclusion was reached.

## J5 — Background load from the simulator

**Preconditions:** the service has been running for at least one minute with no user input.

**Steps:**
1. Leave the App UI tab open without starting anything.

**Expected:**
- `RequestSimulator` drips a canned topic from `session-requests.jsonl` every thirty seconds.
- New sessions appear and conclude on their own; the canned set includes both consensus and max-turns cases, so both outcomes show up over time.

## J6 — Operator halt pauses new sessions

**Preconditions:** the service is running.

**Steps:**
1. Click Halt in the operator control (or `POST /api/system/halt`).
2. Submit a new session (or wait for the simulator's next tick).
3. Click Resume (or `POST /api/system/resume`).

**Expected:**
- While halted, `GET /api/system/status` returns `{ "halted": true }`, the Start button is disabled, and no new session starts — queued requests are skipped by `SessionRequestConsumer`.
- Any session already in flight keeps running to its conclusion; halt gates starts only.
- After Resume, new sessions start again on the next request or simulator tick.
