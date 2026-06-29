# User journeys — react-chatbot

## J1 — Send a question that requires a tool call

**Preconditions:** Service running on declared port (`http://localhost:9524/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9524/` → App UI tab.
2. Click **New conversation**. Enter any title and a user ID. Click **Start**.
3. In the message input, type "Where is my order ORD-003?" and click **Send**.

**Expected:**
- A new turn card appears in the right pane with status `PROCESSING` within 1 s.
- Within 30 s, the turn reaches `COMPLETED`. The reply bubble contains a coherent answer about the order.
- The **Reasoning** section (collapsed) shows at least one `ToolCall` entry with `toolName = get_order_status`, an `inputSummary` referencing `ORD-003`, and a non-empty `resultSummary`.
- The SSE stream delivers the `COMPLETED` transition; the UI updates without a page refresh.

## J2 — Guardrail blocks a policy-violating reply

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `chat-turn.json` includes deliberately policy-violating entries (system-prompt disclosure language; a scope-breach keyword).

**Steps:**
1. Start a new conversation and send any message. Repeat until the third send (the mock selects a violating entry on the first iteration of every third turn, by seed).
2. Watch the turn lifecycle in the network panel (`/api/conversations/sse`).

**Expected:**
- The third turn's first agent iteration produces a policy-violating candidate reply.
- The `before-agent-response` guardrail rejects it. The violating content NEVER lands in `ConversationEntity` — there is no `ReplyRecorded` event with the violating payload.
- The agent loop retries on iteration 2 (or 3 if needed) and produces a clean reply. The turn transitions to `COMPLETED` with a reply that passes all three guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code (`policy-violation | pii-leak | scope-breach`).

## J3 — Three-turn conversation retains full history

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Start a new conversation.
2. Send message 1: "What is your return policy for electronics?"
3. Wait for `COMPLETED`.
4. Send message 2: "What about headphones specifically?"
5. Wait for `COMPLETED`.
6. Send message 3: "And how do I start a return?"
7. Wait for `COMPLETED`.

**Expected:**
- The right pane shows three user bubbles and three reply bubbles in chronological order.
- `GET /api/conversations/{id}` returns a `turns` array with exactly three entries, each with `status = COMPLETED`.
- Each turn has a non-empty `reply.content` and a non-empty `reply.toolCallTrace`.
- The conversation-level `status` is `ACTIVE` throughout.

## J4 — Concurrent message rejected while turn is PROCESSING

**Preconditions:** Service running. Any model provider. The model is slow enough (or mock LLM is used with an artificial delay) that a turn stays in `PROCESSING` for at least 2 s.

**Steps:**
1. Start a new conversation.
2. Send a message. While the turn is still `PROCESSING`, immediately send a second message.

**Expected:**
- The second `POST /api/conversations/{id}/messages` returns `409 Conflict` with a JSON body identifying the already-PROCESSING turn.
- The UI shows an inline "Wait for the current reply" tooltip and does not create a second turn.
- The first turn completes normally; the conversation state is not corrupted.

## J5 — Tool call with no matching data returns a graceful reply

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Start a new conversation.
2. Send the message "What is the status of order ORD-999?"

**Expected:**
- The agent calls `get_order_status("ORD-999")`.
- The tool returns a not-found result (ORD-999 is not in the stub data).
- The reply bubble explains that the order could not be found and suggests the user verify the order ID or contact support. The reply does not invent a status.
- The `Reasoning` section shows the `get_order_status` tool call with a `resultSummary` indicating no result was found.
