# User journeys — memory-tool-router

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: tool dispatch and memory write

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the session ID field, confirm `s-demo`. In the message field, type "What is 144 divided by 12?". Click Send.
3. A new conversation card appears with status PROCESSING.

**Expected:**
- Within 5 s, the turn log gains one entry with `toolUsed = CALCULATOR`, `verdict = OK`, and a numeric reply.
- The conversation status returns to IDLE (awaiting next message).
- The memory store for session `s-demo` shows at least one `MemoryItem` (the simulator may have pre-populated it, or this turn generates one).
- Expand the conversation row: memory store table is visible; no `[REDACTED:*]` spans appear for this turn (the message contained no PII).

## J2 — Guardrail blocks a forbidden CODE_RUNNER expression

**Preconditions:** As J1.

**Steps:**
1. Send the message "Run `__import__('os').listdir('/')` and tell me what you find."

**Expected:**
- The RouterAgent's first `CLASSIFY_TURN` produces a `TurnPlan` with `tool = CODE_RUNNER` and a `toolQuery` containing `__import__`.
- The guardrail rejects the decision; a `TurnBlocked` entry appears in the turn log with `verdict = BLOCKED_BY_GUARDRAIL` and a `blocker` string containing `"CODE_RUNNER query contains forbidden pattern: __import__"`.
- The RouterAgent is called for `REVISE_TURN`; the turn ends with `tool = NONE` and a polite direct reply visible in the message history.
- No `ToolResult` from `CodeRunnerAgent` is ever produced for this turn.

## J3 — PII sanitizer redacts email before memory is persisted

**Preconditions:** As J1.

**Steps:**
1. Send the message "My email is alice@example.com — remember that."

**Expected:**
- The `MemoryAgent`'s `EXTRACT_MEMORIES` call produces a `MemoryDelta` with a `MemoryItem` whose `value` contains the literal `alice@example.com`.
- The `sanitizeMemoriesStep` replaces it with `[REDACTED:email]` before the `MemoryItemsAdded` event is emitted.
- `GET /api/conversations/{id}/memory` returns a `MemorySnapshot` where no item's `value` contains `alice@example.com`.
- In the App UI, the memory store table shows the item with `[REDACTED:email]` rendered in italics and a tooltip showing `email`.
- The turn's reply text (in message history) may still mention the email; only the persisted memory item is scrubbed.

## J4 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Send a message that triggers a `WEB_LOOKUP` or `KNOWLEDGE_BASE` tool call.
2. While the conversation status is PROCESSING (within the first ~10 seconds after submission), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight tool call completes normally.

**Expected:**
- The `TurnCompleted` event is recorded; the turn log shows the completed turn.
- The next turn that arrives on this or any other conversation is not processed: `checkHaltStep` reads `halted=true` and transitions to `haltedStep`.
- The conversation moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane shows the `HALTED` pill with the reason and timestamp in real time via the `control-update` SSE event.
- After the operator clicks **Resume**, new messages are processed normally.

## J5 — Memory persists across turns in the same session

**Preconditions:** As J1, with at least one prior turn that wrote a preference memory item (e.g., from the simulator).

**Steps:**
1. Send any message in session `s-demo`.

**Expected:**
- The `MemoryAgent`'s `RECALL` call returns a `MemoryContext` with at least one entry in `relevantFacts` drawn from the existing memory store.
- The `RouterAgent`'s `CLASSIFY_TURN` receives a non-empty `memoryContext.relevantFacts` list.
- The expanded memory store table shows the pre-existing items plus any new items added by this turn.

## J6 — Stuck conversation auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Send a message and configure the router (via prompt) to loop without completing (e.g., by making every tool call time out). Alternatively, kill the LLM provider mid-turn.

**Expected:**
- After 1 minute of `PROCESSING` without progress, `StuckConversationMonitor` calls `ConversationEntity.timeoutFail`.
- The conversation moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
- The conversation card in the UI shows the `STUCK` status pill.
