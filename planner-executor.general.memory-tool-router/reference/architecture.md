# Architecture — memory-tool-router

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ConversationEndpoint` is the entry point. A message submission writes a `MessageSubmitted` event to `MessageQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts or resumes a `ConversationWorkflow` turn per message. The workflow drives the recall → route-and-guard → execute → sanitize-memory → store-memory → reply loop.

At the start of each turn `MemoryAgent` reads the session's `MemoryEntity` and returns a `MemoryContext`. The `RouterAgent` then classifies the turn into a `TurnPlan` and the dispatch guardrail vets the routing decision before any tool-executor agent is called. The matched tool-executor agent — `CalculatorAgent`, `KnowledgeBaseAgent`, `WebLookupAgent`, or `CodeRunnerAgent` — runs and returns a `ToolResult`. `MemoryAgent` then extracts new facts; the `PiiScrubber` sanitizes them; the sanitized items are written to `MemoryEntity`.

Every conversation transition emits an event on `ConversationEntity`; `ConversationView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `SessionSimulator` drips sample turns for demo purposes; `StuckConversationMonitor` ticks every 30 s to mark long-running `PROCESSING` conversations as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag after each turn.

## Interaction sequence

The sequence diagram traces the happy path (J1). The turn in the middle is the recall → route → guard → execute → extract → sanitize → store → reply cycle. Each iteration ends with a halt-flag check; on `halted=false` the workflow waits for the next message. The four tool-executor agents are condensed into a single participant for readability; in code the workflow's `executeStep` switches on `RoutingDecision.tool`.

## State machine

`ConversationEntity` has five states. `IDLE` is the start state — the conversation exists but is waiting for the next message. `PROCESSING` is the active turn state — most events fire here without changing the status. A conversation lands in `HALTED` (operator paused; resumes when the operator clicks Resume), `STUCK` (no progress for 4 minutes), or `ENDED` (conversation explicitly closed). Unlike the task blueprint, `HALTED` is not terminal — a conversation can resume from `HALTED` once the operator clears the halt flag.

## Entity model

`ConversationEntity` is the source of truth for each conversation; nine event types cover its full lifecycle. `MemoryEntity` is the per-session persistent memory store — keyed by `sessionId`, not `conversationId`, so sessions that span multiple conversations share one store. `SystemControlEntity` carries the operator halt flag. `MessageQueue` is the audit log of submissions. `ConversationView` is the only read-side projection — the UI never queries entities directly.

## Concurrency and timeouts

- Per-step timeouts: `recallStep` 30 s, `routeStep` 45 s, `executeStep` 90 s, `extractMemoriesStep` 30 s, `replyStep` 30 s.
- Tool routing: `RoutingDecision.tool = NONE` bypasses `executeStep` entirely and falls through to `extractMemoriesStep` with an empty `ToolResult`.
- Halt poll: synchronous read of `SystemControlEntity` after each full turn cycle; no caching. A halt arriving during `executeStep` lets the in-flight tool call finish.
- Memory isolation: sessions are separated by `sessionId`; two conversations sharing a session see the same memory store.
- PII scrubber: `PiiScrubber.scrub` is pure and deterministic — same input always yields same output.
- Idempotency: `(sessionId, text)` over a 10 s window deduplicates `POST /api/conversations`.
- Stuck detection: `StuckConversationMonitor` every 30 s; conversations `PROCESSING` for > 4 minutes are marked `STUCK`.
