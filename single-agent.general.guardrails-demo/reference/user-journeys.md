# User journeys — guardrails-demo

## J1 — Send an allowed-topic message and receive a reply

**Preconditions:** Service running on declared port (`http://localhost:9228/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9228/` → App UI tab.
2. From the **Demo scenario** dropdown, pick `Allowed topic`.
3. The message textarea is pre-filled with a general product question.
4. Click **Send**.

**Expected:**
- A new session is created (or the existing session is reused if one is open). A turn card appears in the timeline with status `RECEIVED` within 1 s.
- The turn transitions to `CHECKING_INPUT`. The topic-check badge shows `ALLOWED` (green).
- The turn transitions to `GENERATING`. Within 30 s it reaches `COMPLETED`.
- The turn card shows the agent's reply text, a green `1 iteration` content-policy badge, and the `completedAt` timestamp.
- No `TurnBlocked` event appears in the session's SSE stream.

## J2 — Blocked topic — LLM never called

**Preconditions:** Service running. Any model provider (the topic guardrail runs before the model).

**Steps:**
1. From the **Demo scenario** dropdown, pick `Blocked topic`.
2. The message textarea is pre-filled with a financial-advice request.
3. Click **Send**.

**Expected:**
- A turn card appears in the timeline with status `CHECKING_INPUT` within 1 s.
- The turn transitions immediately to `BLOCKED`. The topic-check badge shows `BLOCKED` (red) with the matched topic name (`financial-advice`).
- The turn card shows the note "LLM not called — topic blocked." No reply section is rendered.
- The service log contains a `topic-blocked` event with the matched keyword but does NOT contain any LLM-call log entry for this turn.
- Fetching `GET /api/sessions/{sessionId}` and reading the blocked turn's `topicCheck.matchedTopic` returns `"financial-advice"` and `reply` is `null`.

## J3 — Content-policy guardrail forces a retry

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `reply-to-message.json` includes deliberately non-compliant entries.

**Steps:**
1. From the **Demo scenario** dropdown, pick `Content-policy retry`.
2. The message textarea is pre-filled with a message whose turn index (modulo seed) triggers a non-compliant mock reply on the first iteration.
3. Click **Send**.

**Expected:**
- The turn passes the topic check (ALLOWED).
- The agent's first candidate reply contains a disallowed phrase. The `before-agent-response` guardrail rejects it. The non-compliant text NEVER appears in the session entity — there is no `ReplyRecorded` event with the bad payload.
- The agent loop retries (iteration 2). The second candidate passes. `ReplyRecorded` is emitted.
- The turn card shows the reply text and a yellow `2 iterations` content-policy badge.
- The service log contains one `guardrail.reject` line naming the violated check (`disallowed-phrase`) and the excerpt that triggered it.

## J4 — Blocked message body absent from LLM call log

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider.

**Steps:**
1. Submit the message "What prescription medication should I take for a headache?" (matches `prescription-medication` blocked topic).
2. Wait for the turn to reach `BLOCKED`.
3. Inspect the service log for any LLM call body entries.
4. Fetch `GET /api/sessions/{sessionId}` and read the blocked turn's `topicCheck`.

**Expected:**
- The service log contains NO LLM call body for this turn. The `debug:agent.task.instructions` entry does not appear.
- The session row shows `topicCheck.outcome = "BLOCKED"` and `topicCheck.matchedTopic = "prescription-medication"`.
- `reply` is `null` on the turn record.
- The blocked message text IS visible on the turn card in the UI — the session log retains it for audit purposes, but it was never passed to the model.

## J5 — Two consecutive turns in the same session

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Open a new session (click `New session`).
2. Send an allowed-topic message (e.g., "What are your support hours?"). Wait for `COMPLETED`.
3. Send a follow-up message that references the prior reply (e.g., "And do you offer weekend support?"). Wait for `COMPLETED`.

**Expected:**
- Both turns appear in the session timeline under the same `sessionId`.
- The second turn's `context.txt` attachment (visible in debug logs) contains a summary of the first turn.
- The agent's second reply reflects awareness of the prior exchange (e.g., references "weekend" specifically rather than restating general support hours).
- Both turns show status `COMPLETED` with `contentPolicyIterations = 1`.
- `GET /api/sessions/{sessionId}` returns a session with `turns` array of length 2.
