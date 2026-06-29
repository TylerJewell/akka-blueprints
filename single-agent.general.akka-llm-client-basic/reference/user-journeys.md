# User journeys — akka-llm-client-basic

## J1 — Send a factual prompt and get a reply

**Preconditions:** Service running on declared port (`http://localhost:9533/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9533/` → App UI tab.
2. From the **Seeded prompts** dropdown, pick `Factual question`.
3. The textarea fills with: "What is the capital of Iceland, and what is its estimated population?"
4. Click **Send**.

**Expected:**
- A new turn card appears in the history list with status `PENDING` within 1 s.
- Within 20 s the card transitions to `REPLIED`. The right pane shows: the full prompt, a non-empty reply text that names Reykjavik, and a metadata row with non-zero latency.
- The guardrail outcome badge reads `PASSED`.
- `GET /api/conversations/{sessionId}` returns a JSON session whose `turns[0].status` is `"REPLIED"` and `turns[0].reply.text` is non-empty.

## J2 — Guardrail blocks a malformed reply

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `generate-reply.json` includes a deliberately empty-text entry.

**Steps:**
1. Submit any seeded prompt four times in a row (J1 steps × 4).
2. On the fourth submission, watch the SSE stream (`/api/conversations/{sessionId}/sse`) in the browser network panel.

**Expected:**
- The fourth turn's first agent iteration produces a reply with empty `text`.
- `ReplyGuardrail` rejects it. The empty reply NEVER lands as a `TurnReplied` event — no `REPLIED` transition occurs with empty text.
- The agent loop retries on iteration 2 (or 3) and produces a non-empty reply. The card transitions to `REPLIED` with valid text.
- The service log shows a `guardrail.reject` line for the failed iteration naming the `EMPTY_TEXT` check.

## J3 — Empty prompt is rejected before any model call

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI's **Prompt** textarea, clear all text.
2. Click **Send**.

**Expected:**
- The UI receives a `400 Bad Request` response from the endpoint.
- No new turn card appears in the history list.
- The session's turn count is unchanged.
- No LLM call is made (the service log shows no `agent.task.start` line).

## J4 — Multi-turn conversation preserves context

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Send prompt 1: "My name is Alex. What is the capital of Iceland?"
2. Wait for `REPLIED`.
3. Send prompt 2: "What did I say my name was?"

**Expected:**
- Turn 2's agent call includes both prior turns in the formatted context passed as `TaskDef.instructions(...)`.
- The reply to prompt 2 references "Alex" (real LLM) or includes the prior-turn content (mock LLM).
- `GET /api/conversations/{sessionId}` returns a session with `turns` of length 2, both in `REPLIED` status.

## J5 — New session resets history

**Preconditions:** Service running. At least one existing turn in the current session.

**Steps:**
1. Click **New session** in the App UI.
2. Send the same factual prompt as J1.

**Expected:**
- A new `sessionId` is minted (`POST /api/conversations` called).
- The turn history panel shows only the new turn — no prior turns from the previous session.
- `GET /api/conversations` returns at least two sessions. The new session has one turn; the old session's turns are unchanged.

## J6 — Over-length reply is caught by guardrail

**Preconditions:** Service running with the mock LLM selected. The mock's `generate-reply.json` includes an entry where `text` exceeds 8 000 characters.

**Steps:**
1. Submit any seeded prompt until the mock selects the over-length entry (every 4th turn per the mock's seeding logic).

**Expected:**
- The over-length candidate is rejected by `ReplyGuardrail` with the `OVER_LENGTH` check named in the service log.
- The agent loop retries and returns a reply within the length bound.
- The turn reaches `REPLIED` with `reply.text.length() <= 8000`.
