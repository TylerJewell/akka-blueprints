# User journeys — multi-model-chatbot

## J1 — Start a session and get a support reply

**Preconditions:** Service running on declared port (`http://localhost:9994/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9994/` → App UI tab.
2. Click **New session**. Enter display name "Alice T." Leave the default provider.
3. Type a billing inquiry message: "Can you help me understand my invoice from last month?"
4. Click **Send**.

**Expected:**
- The message bubble appears with status `RECEIVED` within 1 s.
- The bubble transitions to `SANITIZED` within 1 s. If no PII was found, no PII chips appear; the sanitized text equals the original.
- Within 30 s the bubble transitions to `REPLIED`. The reply bubble shows `replyText` (non-empty, on-topic), a provider attribution label (e.g., `claude-sonnet-4-6 · anthropic`), and the `repliedAt` timestamp.
- The session card in the left rail updates its last-activity age.

## J2 — Guardrail blocks a policy-violating reply

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `generate-reply.json` includes deliberately violating entries.

**Steps:**
1. Send any message three times within the same session (using the same mock seed window).
2. Watch the third turn's lifecycle in the network panel via `/api/chat/{sessionId}/sse`.

**Expected:**
- The third turn's first agent iteration produces a policy-violating reply.
- The `before-agent-response` guardrail rejects it. The violating reply NEVER lands in `ConversationEntity` — there is no `ReplyGenerated` event with the violating payload.
- The agent loop retries on iteration 2 and produces a clean reply. The bubble transitions to `REPLIED` with content that passes all guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed (e.g., `harmful-content-pattern` or `pii-re-exposure`).

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Send a message containing `alice@corp.com`, `555-867-5309`, and `Account: ACC-1234567`.
2. Wait for `REPLIED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).
4. Fetch `GET /api/chat/{sessionId}` and read the turn's `userText`.

**Expected:**
- The logged LLM call body contains only redacted forms: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-ACCOUNT-ID]`. The raw strings do not appear.
- The turn's `userText` in the JSON still contains the raw strings — the entity preserves them for audit.
- `piiCategoriesFound` lists `email`, `phone`, `account-id` (at minimum).

## J4 — Provider switching mid-session

**Preconditions:** Service running. At least two providers configured (or mock LLM mode, which simulates provider selection).

**Steps:**
1. Start a new session. Send one message. Note the provider attribution on the reply bubble.
2. Click the `activeProvider` chip in the session header. Select a different provider.
3. Send a second message in the same session.

**Expected:**
- The second reply bubble shows a different provider attribution label than the first.
- The first reply bubble still shows the original provider name unchanged.
- `GET /api/chat/{sessionId}` confirms `activeProvider` matches the selected value and that each turn's `reply.providerName` reflects the provider active at the time of that turn.

## J5 — FAILED turn displays correctly

**Preconditions:** Mock LLM mode. All 3 guard iterations exhausted by configuring the mock to always return a violating reply for a specific seed window.

**Steps:**
1. Send a message in a session whose seed lands on the always-violating mock window.
2. Wait for the turn to reach `FAILED`.

**Expected:**
- The chat thread shows a red error bubble for that turn with a human-readable failure reason.
- The session remains `ACTIVE`; subsequent messages in the same session can succeed.
- `GET /api/chat/{sessionId}` returns the turn with `status: "FAILED"` and a non-empty `reason` (surfaced from the workflow's error step).

## J6 — Conversation history is passed to subsequent turns

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Start a new session. Send three messages on a single topic (e.g., a multi-step billing inquiry).
2. In the fourth message, refer to a detail from turn 1 (e.g., "As I mentioned earlier, the charge was in April.").

**Expected:**
- The reply to turn 4 references the prior context appropriately — demonstrating the workflow sends full conversation history to the agent.
- Each turn in `GET /api/chat/{sessionId}` shows its own `sanitizedText`, `piiCategoriesFound`, and `reply` independently.
