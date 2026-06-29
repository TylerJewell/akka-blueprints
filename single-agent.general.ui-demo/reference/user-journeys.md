# User journeys — ui-demo

## J1 — Submit a seeded prompt and get a streaming response

**Preconditions:** Service running on declared port (`http://localhost:9352/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9352/` → App UI tab.
2. Click `+ New session`. A session card appears in the left rail with status `IDLE`.
3. From the **Prompt** dropdown, select "What is event sourcing and why would I use it?".
4. Click **Send**.

**Expected:**
- The turn status transitions to `STREAMING` within 1 s. The `Send` button is disabled.
- The agent bubble starts filling with streamed text. Citation markers `[1]`, `[2]` appear inline as they arrive.
- Within 30 s the stream closes. The turn reaches `TURN_COMPLETE`. The `RESPONSE_COMPLETE` envelope lands.
- The Citations section expands beneath the bubble showing every cited source with its snippet.
- A `Suggested follow-up` chip appears. Clicking it pre-fills the prompt textarea.
- The session status returns to `IDLE`. The `Send` button re-enables.

## J2 — Guardrail blocks a malformed response

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-prompt.json` includes deliberately malformed entries (one with a blank `answer`; one with a `citations[].source` that is an empty string).

**Steps:**
1. Click `+ New session`.
2. Submit three prompts in a row using any seeded text (J1 steps × 3, without closing the session between turns).
3. On the fourth turn, the mock selects a malformed entry (every 4th turn, per the mock's seed logic).
4. Watch the session's SSE stream in the browser network panel.

**Expected:**
- The fourth turn's first agent iteration produces a malformed `CopilotResponse`.
- The `before-agent-response` guardrail rejects it. The malformed payload NEVER lands in `SessionEntity` — there is no `TurnCompleted` event with the malformed answer.
- The agent loop retries on iteration 2 and produces a well-formed response. The chat bubble eventually shows the valid answer.
- The service log contains one `guardrail.reject` line per rejected iteration, naming which check failed (e.g., `answer-blank` or `citation-source-empty`).
- The UI never flashes the malformed payload; the chat bubble shows the pulsing ellipsis throughout the retry.

## J3 — Session history persists across page reloads

**Preconditions:** Service running. At least one session with two or more completed turns exists (run J1 twice in the same session).

**Steps:**
1. Note the `sessionId` from the session card in the left rail.
2. Hard-refresh the browser page (Ctrl+Shift+R / Cmd+Shift+R).
3. Locate the same session card in the left rail and click it.

**Expected:**
- The right pane loads the full turn history in order: each user bubble (prompt text) followed by the corresponding agent bubble (answer + citations + suggested follow-up chip).
- No turns are missing or duplicated.
- Fetching `GET /api/sessions/{sessionId}` returns the same turns in the same order.

## J4 — Over-length response caught by guardrail

**Preconditions:** Mock LLM mode. The `answer-prompt.json` mock includes one entry whose `tokenCount` is 3000, exceeding the default ceiling of 2048.

**Steps:**
1. Click `+ New session`.
2. Select the seeded prompt whose mock maps to the over-length entry (documented in `src/main/resources/mock-responses/answer-prompt.json` comments).
3. Click **Send** and observe the turn lifecycle.

**Expected:**
- The first agent iteration returns a `CopilotResponse` with `tokenCount = 3000`.
- The `ResponseGuardrail` rejects it with `token-count-exceeded`. The rejection appears in the service log.
- The agent retries on iteration 2 with a shorter answer (`tokenCount` within the ceiling).
- The chat bubble shows the shorter, well-formed answer. The session reaches `TURN_COMPLETE`.

## J5 — Follow-up chain builds coherent history

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Click `+ New session`.
2. Submit the seeded prompt "What is event sourcing and why would I use it?". Wait for `TURN_COMPLETE`.
3. Click the **Suggested follow-up** chip that appears beneath the first answer.
4. Click **Send** again.

**Expected:**
- The prompt textarea pre-fills with the suggested follow-up question.
- The second turn's agent call receives both the original prompt and the first answer as conversation context (visible in the service log's agent task instructions).
- The second answer references the first turn's concepts without re-explaining them from scratch.
- The session card in the left rail shows `turnCount = 2`.
