# User journeys — mcp-chat-server

## J1 — Send a message and receive a tool-augmented reply

**Preconditions:** Service running on declared port (`http://localhost:9818/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9818/` → App UI tab.
2. Click **New conversation**, enter a title (e.g., `"Akka routing docs"`), click **Create**.
3. In the message textarea, type `"Search for Akka HTTP routing DSL documentation."`.
4. Click **Send**.

**Expected:**
- The new turn card appears in the thread with status `RECEIVED` within 1 s.
- The card transitions to `RUNNING` within 1 s (animated pill).
- Within 30 s the card reaches `REPLIED`. The agent bubble shows the reply text, a blue `search · ALLOWED` tool-call chip, and an iterations-used badge of `1`.
- The conversation's `lastActiveAt` updates in the left rail.

## J2 — Before-tool-call guardrail blocks a disallowed tool

**Preconditions:** Service running with the mock LLM selected. The mock's `chat-turn.json` includes an entry that instructs the agent to call a tool not on the allow-list (e.g., `"admin-exec"`).

**Steps:**
1. Create a new conversation.
2. Send the message `"Run admin-exec with argument reset"`.
3. Watch the turn card in the thread.

**Expected:**
- The agent attempts to call `admin-exec`; `ToolCallGuardrail` blocks it before any HTTP request leaves the JVM.
- The agent bubble shows an amber `admin-exec · BLOCKED` chip in `toolCallsAudited`.
- The reply text explains that the tool was unavailable and offers what it can from context alone (e.g., `"The admin-exec tool is not available in this environment. I can answer questions about the operation you are trying to perform."`).
- The turn reaches `REPLIED` — it is NOT `FAILED`. The blocked tool call did not abort the task.
- The service log shows one `guardrail.block-tool-call` line naming the failed check (`tool-name-not-on-allow-list`).

## J3 — Before-agent-response guardrail rejects and retries

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `chat-turn.json` includes a deliberately malformed entry whose `replyText` exceeds `configuredMaxReplyChars`.

**Steps:**
1. Create a new conversation.
2. Send a message three times in a row — the third submission's mock entry is the malformed one (deterministic by seed).
3. Watch the third turn's lifecycle in the network panel (`/api/conversations/{id}/sse`).

**Expected:**
- The third turn's first agent iteration produces a reply that exceeds the length limit.
- `ReplyGuardrail` rejects it. The malformed reply NEVER lands in `ConversationEntity` — there is no `ReplyRecorded` event with the oversized payload.
- The agent loop retries on iteration 2 and produces a valid reply. The turn reaches `REPLIED` with `iterationsUsed = 2`.
- The service log shows one `guardrail.reject` line naming the failed check (`reply-text-exceeds-max-chars`).

## J4 — Two conversations run concurrently and remain isolated

**Preconditions:** Service running with either the mock LLM or a real provider.

**Steps:**
1. Open two browser tabs at `http://localhost:9818/`.
2. In tab A, create conversation `"Alpha"` and send message `"What time is it?"`.
3. Immediately in tab B, create conversation `"Beta"` and send message `"Search for Akka clustering docs"`.
4. Wait for both conversations to reach `REPLIED`.

**Expected:**
- The `Alpha` conversation's reply references the time (via the `time` tool) with no mention of clustering.
- The `Beta` conversation's reply references Akka clustering documentation (via the `search` tool) with no mention of time.
- Each conversation's turn list in the left rail shows exactly one turn. There is no cross-contamination of history or tool-call audit records.
- Both `ConversationEntity` instances exist independently in the event log — each emits its own `ConversationCreated`, `MessageReceived`, `AgentRunStarted`, and `ReplyRecorded` events with distinct `conversationId` values.

## J5 — Graceful shutdown drains in-flight requests

**Preconditions:** Service running with a real model provider (so the agent actually takes a few seconds to reply).

**Steps:**
1. Send a message in an open conversation.
2. Immediately (within 2 s) stop the service via Ctrl+C or the Akka MCP stop command.

**Expected:**
- The in-flight `ChatWorkflow.runAgentStep` is given up to its step timeout (60 s) to complete before the JVM exits.
- If the step completes within the drain window, the turn reaches `REPLIED` and the entity event is persisted before shutdown.
- If the drain window expires, the workflow's recovery will replay the step on the next startup, and the turn eventually reaches `REPLIED` or `FAILED` with a clear failure reason.
- The service log shows a `shutdown-initiated` line followed by either `drain-completed` or `drain-timeout` — never a silent kill.
