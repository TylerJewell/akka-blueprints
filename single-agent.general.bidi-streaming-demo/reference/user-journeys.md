# User journeys — bidi-demo

## J1 — Open a channel and receive a streamed reply

**Preconditions:** Service running on `http://localhost:9799/`; a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9799/` → App UI tab.
2. Click **Open channel**. Accept the auto-generated channel name and default turn budget of 10. Click **Create**.
3. The new channel card appears in the left rail with status `OPEN`.
4. Type the message "What is bidirectional streaming?" in the message input and click **Send**.

**Expected:**
- The message posts and the turn card appears in the right pane with status `PROCESSING` within 1 s.
- The channel status changes to `ACTIVE`.
- Within 15 s, response frames appear one by one in the turn card. Each frame shows its `frameIndex` and `content`. A pulsing dot is visible while more frames are incoming.
- When the `done == true` frame arrives, the pulsing dot stops and the turn-status badge changes to `COMPLETE`.
- The message input re-enables and the turn count in the channel card increments to `1 / 10`.

## J2 — Guardrail blocks a malformed frame list

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `respond-to-message.json` includes a deliberately malformed entry (a frame with empty `content` and `done == false`).

**Steps:**
1. Open a channel and send messages until the mock returns the malformed entry (every 3rd call in the mock's rotation).
2. Watch the turn lifecycle in the browser's network panel (`/api/channels/{id}/sse`).

**Expected:**
- The malformed frame list is rejected by `FrameGuardrail` before any frame is written to `ChannelEntity`. No `FramePublished` event appears in the SSE stream during the rejected iteration.
- The agent loop retries on iteration 2. The retry produces a valid frame list.
- The turn reaches `COMPLETE` with only valid frames visible in the UI.
- The service log shows one `guardrail.reject` line for the rejected iteration, naming the check that failed (e.g., `content-empty-before-done`).

## J3 — Turn budget exhaustion closes the channel

**Preconditions:** Service running. Open a channel with `turnBudget = 3`.

**Steps:**
1. Open a channel with turn budget set to 3.
2. Send three messages, waiting for each turn to reach `COMPLETE` before sending the next.

**Expected:**
- After the third turn completes, `flushStep` detects `turns.size() >= turnBudget` and calls `ChannelEntity.closeChannel("turn-budget-exhausted")`.
- The channel status changes to `CLOSED`.
- The SSE stream emits a `lifecycle` event with `event: ChannelClosed` and `reason: "turn-budget-exhausted"`.
- The message input in the UI is replaced by the "Turn budget exhausted" banner.
- A fourth attempt to POST to `/api/channels/{id}/messages` returns `409 Conflict`.

## J4 — Two subscribers see the same frames

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Open a channel in browser tab A.
2. Open a second browser tab B and navigate to the same channel URL.
3. In tab A, send a message.

**Expected:**
- Both tab A and tab B's SSE connections receive the same `frame` events in the same order.
- Each frame's `frameIndex` in both tabs increments correctly from 0 with no gaps.
- No frame is duplicated — each `FramePublished` entity event produces exactly one SSE event per subscriber.
- Tab B, which joined after the channel was opened, receives any frames from earlier turns (replayed from the entity's event log) before receiving the new turn's frames.

## J5 — Multi-turn context accumulation

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Open a channel and send the message "My name is Alex."
2. Wait for `COMPLETE`. Then send "What is my name?"

**Expected:**
- The second turn's response frames reference "Alex" — the agent read the turn history attachment and maintained context across turns.
- The history attachment for the second turn contains the first turn's message and frames (verifiable via `GET /api/channels/{id}` and inspecting `turns[0].frames`).
- The turn count increments to `2 / 10` (or the configured budget).
