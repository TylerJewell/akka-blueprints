# User journeys — gemini-fullstack

## J1 — Create a conversation and receive an agent reply

**Preconditions:** Service running on declared port (`http://localhost:9552/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9552/` → App UI tab.
2. Click **New conversation**. Enter the title "Akka quick question" and confirm.
3. Type "What is an EventSourcedEntity?" in the message input and click **Send**.

**Expected:**
- The conversation appears in the left panel with status `ACTIVE` within 1 s of creation.
- After Send, the status chip transitions to `AWAITING_REPLY` and the "Thinking..." indicator appears in the thread.
- Within 30 s, the agent reply card appears in the thread. The status chip transitions to `REPLY_RECORDED`. The reply has a non-empty `content` string and a non-zero token-count chip.
- The SSE stream (visible in browser devtools → Network → EventStream) shows one `conversation-update` event per state transition.

## J2 — Two conversations remain isolated

**Preconditions:** Service running with any model provider.

**Steps:**
1. Create conversation A: "Topic A". Send the message "Hello from A."
2. Without waiting for a reply, create conversation B: "Topic B". Send the message "Hello from B."
3. Wait for both conversations to reach `REPLY_RECORDED`.
4. Inspect each conversation's message list via `GET /api/conversations/{idA}` and `GET /api/conversations/{idB}`.

**Expected:**
- Conversation A's `messages` array contains only the user message "Hello from A." and its agent reply.
- Conversation B's `messages` array contains only the user message "Hello from B." and its agent reply.
- No cross-contamination: neither array references the other conversation's content.
- The two workflow instances (one per turn) ran independently without interfering.

## J3 — SSE delivers updates without polling

**Preconditions:** Service running. Browser devtools open on the Network panel.

**Steps:**
1. Select an existing conversation in the App UI.
2. Open browser devtools → Network → filter for EventStream / SSE.
3. Send a new message.
4. Watch the devtools panel for network activity during the reply wait.

**Expected:**
- Exactly one open SSE connection (`GET /api/conversations/{id}/sse`) is visible in the Network panel. No repeated `GET /api/conversations/{id}` polling requests appear while waiting for the reply.
- The `AWAITING_REPLY` → `REPLY_RECORDED` status transition arrives as a `conversation-update` SSE event. The UI updates without the user refreshing.
- After the reply lands, the SSE connection remains open (the conversation may receive more messages).

## J4 — Conversation history is durable across page refresh

**Preconditions:** Service running with any model provider. At least one conversation with a completed reply exists.

**Steps:**
1. Note the `conversationId` of a conversation that has reached `REPLY_RECORDED`.
2. Close the browser tab completely.
3. Open a new tab and navigate to `http://localhost:9552/`.
4. Select the same conversation from the left panel (or fetch `GET /api/conversations/{id}` directly).

**Expected:**
- The conversation list shows the same conversations as before the refresh.
- The selected conversation's thread shows all prior turns: both the user message and the agent reply, with their original `createdAt` timestamps.
- No data was lost between the refresh; the `ConversationEntity` event log persisted the history durably.

## J5 — Multi-turn context carries forward

**Preconditions:** Service running. A conversation exists with at least two completed turns (two user messages and two agent replies).

**Steps:**
1. Open a conversation that has already had one complete turn.
2. Send a follow-up message that explicitly references the earlier topic (e.g., "Can you give me a more detailed example of that?").
3. Wait for the reply.

**Expected:**
- The agent reply addresses the follow-up in the context of the earlier exchange — it does not treat the second message as a standalone question with no context.
- `GET /api/conversations/{id}` shows 4 messages in total (2 user, 2 agent) ordered chronologically.
- The `ConversationContextBuilder` included the prior turns in the `conversation.json` attachment (visible in debug logs as `debug:agent.task.attachment` if `LOG_LEVEL=DEBUG` is set).
