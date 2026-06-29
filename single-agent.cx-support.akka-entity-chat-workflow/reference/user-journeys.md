# User journeys — entity-workflow-chat

## J1 — Start a conversation and complete three turns

**Preconditions:** Service running on declared port (`http://localhost:9134/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9134/` → App UI tab.
2. Enter `Duplicate charge on my June statement` in the Topic field and click **New conversation**.
3. In the right pane, type `I was charged twice on June 14th. Can you check that?` and click **Send**.
4. Wait for the status to return to `OPEN`.
5. Send a second message: `Thank you, and when will the refund appear?`
6. Wait for the reply. Send a third message: `Got it, thanks.`

**Expected:**
- Each turn: status transitions to `AGENT_THINKING` within 1 s of Send, then back to `OPEN` within 30 s.
- Every reply appears as an agent bubble with non-empty `replyText`.
- After the third message, `suggestClose` on the `AgentReply` is `true` (mock: the final turn's mock entry has `suggestClose: true`). The `Close` button pulses or gains emphasis.
- `GET /api/conversations/{id}` returns a conversation with `turns` of length 3, all in `COMPLETED` status.

## J2 — History compaction fires and context survives

**Preconditions:** Service running with mock LLM. Token-budget threshold set to a low value (200 chars) via an env override for testing (`COMPACTION_THRESHOLD_CHARS=200`), or a seed conversation with enough turns to exceed 4 000 tokens.

**Steps:**
1. Load the **Billing dispute** seeded scenario (5 turns pre-filled).
2. Click **New conversation** and **Send** the first seeded turn.
3. Continue sending all 5 seeded turns in sequence.
4. Observe the conversation thread in the right pane.

**Expected:**
- After turn 4 or 5 (depending on accumulated size), the status momentarily shows `COMPACTING`, then returns to `OPEN`.
- A compaction marker line appears in the thread between the compacted turns and the current turn, labelled "compacted — N turns summarised". Clicking the expand toggle shows the bullet-list summary.
- `GET /api/conversations/{id}` shows a non-empty `summaries` array with a `ConversationSummary` whose `turnsCompacted >= 3`.
- The next agent reply (turn 5 or 6) references the summarised context correctly — it does not ask about information already confirmed in earlier turns.
- The full turn log is still accessible via the entity; no turn data was deleted, only summarised.

## J3 — PII is stripped before the agent context is assembled

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the agent context attachment is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Start a new conversation with topic `Password reset help`.
2. Send a message containing the literal strings: `My email is jane.doe@example.com`, `my card ends in 4111-1111-1111-1111`, and `my phone is +1-800-555-0199`.
3. Wait for the agent reply.
4. Inspect the service log for the agent context attachment (`debug:agent.task.attachment`).
5. Fetch `GET /api/conversations/{id}` and inspect `turns[0].userMessage`.

**Expected:**
- The logged context attachment contains only `[REDACTED-EMAIL]`, `[REDACTED-PCN]`, and `[REDACTED-PHONE]`. The raw strings do not appear.
- `turns[0].userMessage.redactedText` in the JSON contains the redacted forms.
- `turns[0].userMessage.piiCategoriesFound` lists `email`, `payment-card-number`, `phone` (at minimum).
- The entity's `MessageReceived` event retains the raw text (accessible via event log or by reconstructing from entity audit). The UI never displays it.

## J4 — Workflow genuinely pauses between turns

**Preconditions:** Service running with mock LLM. Browser dev tools open.

**Steps:**
1. Start a new conversation.
2. Send one message and wait for the reply.
3. After the reply appears, wait 15 seconds without sending another message.
4. Watch the SSE stream (`/api/conversations/sse`) in the Network panel.

**Expected:**
- After the reply lands, the conversation status remains `OPEN` and no further SSE events appear.
- No `AgentReplied` or `AGENT_THINKING` events fire during the 15-second wait.
- Sending a second message resumes the workflow within 1 s; `AGENT_THINKING` appears on the SSE stream.
- This confirms the workflow is idle, not polling — it advanced only on the explicit user message.

## J5 — Closing a conversation is terminal

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Start a conversation and send one turn.
2. Click the **Close** button after the reply arrives.
3. Attempt to send another message to the same conversation via `POST /api/conversations/{id}/messages`.

**Expected:**
- After Close, the status pill changes to `CLOSED` and the message input in the right pane is disabled.
- The `POST /api/conversations/{id}/messages` returns `409 Conflict` (or equivalent) — the entity rejects a message to a closed conversation.
- `GET /api/conversations/{id}` shows `status: CLOSED` and a non-null `closedAt` timestamp.
- The full turn log remains intact — the close operation does not delete history.
