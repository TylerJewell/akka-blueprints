# User journeys — with Signals & Queries

## J1 — Start a session and receive the first reply

**Preconditions:** Service running on declared port (`http://localhost:9231/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9231/` → App UI tab.
2. Enter a session name (e.g., `Test session 1`) and an initial prompt (e.g., `What is a workflow signal?`).
3. Enter any value in `Submitted by` and click **Start session**.

**Expected:**
- The new session card appears in the live list with status `STARTING` within 1 s.
- The status transitions to `ACTIVE` within 1 s. The right-pane conversation thread shows turn 1: the prompt plus the agent's reply. The reply is non-empty.
- The turn count badge on the card shows `1`.
- The `Add turn` panel is visible.

## J2 — Inject a follow-up signal mid-session

**Preconditions:** Session from J1 is in `ACTIVE` state.

**Steps:**
1. In the `Your prompt` input, type a follow-up question (e.g., `Can signals carry payloads?`).
2. Click **Send**.

**Expected:**
- A `204` response is returned (no body).
- Within 30 s, a second turn appears in the conversation thread: the follow-up prompt plus a new reply. The agent's reply incorporates the context of the first turn — it does not restart the conversation.
- The session card's turn count increments to `2`.

## J3 — Operator pause, inject corrective note, resume

**Preconditions:** Session from J2 is in `ACTIVE` state.

**Steps:**
1. Click **Pause**. Enter a reason (e.g., `Drift: agent is leaving the Akka scope`).
2. Observe the session card transitions to `PAUSED`.
3. In the `Corrective note` textarea (now visible), type: `Restrict answers to Akka Workflow primitives only.`
4. Click **Send Resume**.

**Expected:**
- After step 2, the session status is `PAUSED`. The `Add turn` panel is hidden; the `Pause` button is disabled; the `Resume` button and corrective note field are visible.
- After step 4, the session transitions back to `ACTIVE`.
- Type one more prompt and click **Send**. The agent's reply begins with an acknowledgment of the operator note (one sentence) before answering the prompt. The corrective note itself appears as an `OPERATOR` source turn in the conversation thread.

## J4 — Guardrail rejects a blank signal

**Preconditions:** Session is in `ACTIVE` state.

**Steps:**
1. In the `Your prompt` input, type a single space character (or leave it blank if the input trims whitespace).
2. Click **Send**.

**Expected:**
- The API returns `400` with body `{ "rejectedCheck": "prompt-empty", "detail": "Prompt must be non-empty and non-whitespace." }`.
- The error detail appears inline in red beneath the `Your prompt` input — no browser alert, no page reload.
- The session's turn count does not change. No `TurnAdded` event appears in the service log for this attempt.
- The session remains `ACTIVE`.

## J5 — Query session state without triggering a turn

**Preconditions:** Session is in `ACTIVE` state with at least 1 turn.

**Steps:**
1. Click the **Query State** button in the session detail header.
2. Click it again immediately (three times total within a short interval).

**Expected:**
- Each click returns the current `SessionState` within 100 ms.
- The turn count does not change after any of the clicks. The session log on the entity has no new events added — `getSessionQuery` is side-effect-free.
- The detail pane refreshes with the same data each time (no new turns, same status, same turn list).

## J6 — Close a session

**Preconditions:** Session from J1–J3 is in `ACTIVE` or `PAUSED` state.

**Steps:**
1. Click **Close** in the operator panel.

**Expected:**
- The session card transitions to `CLOSING` then `CLOSED` within 2 s.
- The `Add turn` panel disappears. The `Pause`, `Resume`, and `Close` buttons disappear.
- Calling `PUT /api/sessions/{id}/signal/turn` now returns `409 Conflict` (the workflow is terminated; signals can no longer be delivered).
- The turn history remains visible in the conversation thread.
