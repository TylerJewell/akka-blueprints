# User journeys — voice-agent

## J1 — Start a session and receive a spoken reply

**Preconditions:** Service running on declared port (`http://localhost:9873/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9873/` → App UI tab.
2. From the **Scenario** dropdown, pick `Billing Query`.
3. Click **Load seeded example** to fill the Caller ID and Transcript fields.
4. Click **Send audio**.

**Expected:**
- The new session card appears in the live list with status `AUDIO_RECEIVED` within 1 s.
- The card transitions to `TRANSCRIPT_SANITIZED` within 1 s. The right-pane detail shows the redacted transcript; PII category chips appear if the seed transcript contains any PII.
- Within 30 s the turn reaches `REPLY_RECORDED`. The reply text is visible in the turn block with a non-empty `replyText` (≤ 500 chars), a valid voice-tone badge (WARM / NEUTRAL / FIRM), and a topic classification chip (`billing`).
- Within 1 s of `REPLY_RECORDED`, the turn reaches `AUDIO_SYNTHESISED`. A "Play audio" button appears and, when clicked, plays the stub WAV.

## J2 — Guardrail blocks a non-compliant reply

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `voice-respond.json` includes two deliberately malformed entries — one with `replyText` over 500 characters, one with a `tone` value outside the enum.

**Steps:**
1. Send any seeded scenario three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/conversations/sse`).

**Expected:**
- The third turn's first agent iteration produces a non-compliant reply.
- The `before-agent-response` guardrail rejects it. The non-compliant reply NEVER lands in `ConversationEntity` — there is no `ReplyRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a compliant reply. The turn transitions to `REPLY_RECORDED` with a reply that passes all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration naming which check failed.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a transcript containing the literal strings `My name is Sarah Jones`, `call me on 07700-900-123`, and `account number 55667788`.
2. Wait for `REPLY_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/conversations/{id}` and read `turns[0].request.rawTranscript`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-NAME]`, `[REDACTED-PHONE]`, `[REDACTED-ACCOUNT]`. The raw strings do not appear.
- `turns[0].request.rawTranscript` in the JSON still contains the raw strings — the audit log preserves them.
- `turns[0].sanitized.piiCategoriesFound` lists `name`, `phone`, `account-number` (at minimum).

## J4 — Multi-turn session accumulates correct history

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Start a new session with the `Technical Support` seeded scenario (J1 steps). Wait for `AUDIO_SYNTHESISED`.
2. In the right pane, note the `conversationId`.
3. Click **Add turn** and submit a follow-up transcript: "The error code I'm seeing is E-404. It started this morning."
4. Wait for the second turn to reach `AUDIO_SYNTHESISED`.

**Expected:**
- The session card in the left rail shows `2 turns`.
- The right pane timeline shows both turns in chronological order: turn 1 above turn 2.
- The second turn's agent call received the first turn's exchange in its conversation history (the `instructions` field in the task). The reply in turn 2 references or acknowledges the context from turn 1 (observable in real-LLM mode; in mock mode the reply is seeded but the history field in the task log is still populated).
- Each turn has its own distinct `turnId`, status pill, sanitized transcript block, and reply section.

## J5 — Escalation flag surfaces in the UI

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a cancellation-request transcript that explicitly asks to speak with a manager: "I want to cancel and I need to speak to a manager right now."
2. Wait for `REPLY_RECORDED`.

**Expected:**
- The reply's `escalationFlag` is `true`.
- The turn block in the UI displays an escalation flag badge ("Needs human").
- The `replyText` acknowledges the escalation request and communicates a handoff — it does not refuse or deflect.
- The session remains in `ACTIVE` status so a supervisor can read the context before taking the call.
