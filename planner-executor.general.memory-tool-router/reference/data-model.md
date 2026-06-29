# Data model — memory-tool-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MessageRequest` | `sessionId` | `String` | no | Session identifier; groups conversations sharing one memory store. |
| | `text` | `String` | no | User-submitted message text. |
| | `sentBy` | `String` | no | UI identifier of the submitter. |
| `MemoryItem` | `key` | `String` | no | Short camelCase or kebab-case identifier for this memory entry. |
| | `value` | `String` | no | One-sentence declarative value (may contain redaction tags after sanitize step). |
| | `kind` | `MemoryKind` | no | FACT / PREFERENCE / ENTITY_MENTION / CORRECTION. |
| | `recordedAt` | `Instant` | no | When this item was written to the memory store. |
| `MemorySnapshot` | `sessionId` | `String` | no | Session the snapshot belongs to. |
| | `items` | `List<MemoryItem>` | no | Full current memory store. |
| `MemoryContext` | `sessionId` | `String` | no | Session for which the context was produced. |
| | `relevantFacts` | `List<String>` | no | Up to 5 facts selected from the memory store for this turn. |
| | `totalMemoryItems` | `int` | no | Total items in the store, including non-selected ones. |
| `MemoryDelta` | `toAdd` | `List<MemoryItem>` | no | New items to write (pre-sanitize; may contain raw PII). |
| | `toForget` | `List<String>` | no | Keys of items to remove from the store. |
| `RoutingDecision` | `tool` | `ToolKind` | no | Which tool-executor agent runs this turn. |
| | `toolQuery` | `String` | no | The query or expression passed to the tool. |
| | `rationale` | `String` | no | One-sentence justification for the routing choice. |
| `TurnPlan` | `routingDecision` | `RoutingDecision` | no | The routing decision for this turn. |
| | `memoryUpdates` | `List<MemoryItem>` | no | New memory items proposed by the router (pre-sanitize). |
| `ToolResult` | `tool` | `ToolKind` | no | Tool that ran (or was called with NONE). |
| | `query` | `String` | no | Echo of the tool query. |
| | `ok` | `boolean` | no | True if the tool-executor could fulfill the query. |
| | `content` | `String` | no | Raw textual result (pre-sanitize for memory extraction). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `TurnEntry` | `turnNumber` | `int` | no | 1-based turn counter within this conversation. |
| | `messageText` | `String` | no | The user message that started this turn. |
| | `toolUsed` | `ToolKind` | no | Tool dispatched (or NONE). |
| | `verdict` | `TurnVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `reply` | `String` | no | The assistant's reply text produced this turn. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `completedAt` | `Instant` | no | When the turn completed. |
| `TurnLog` | `entries` | `List<TurnEntry>` | no | Append-only per-conversation turn log. |
| `Conversation` (entity state) | `conversationId` | `String` | no | Unique id. |
| | `sessionId` | `String` | no | Session this conversation belongs to. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `messageHistory` | `List<String>` | no | Alternating user/assistant messages, oldest first. |
| | `turnLog` | `Optional<TurnLog>` | yes | Populated after first `TurnCompleted` or `TurnBlocked`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `ConversationFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `ConversationHaltedOperator` or `ConversationHaltedAutomatic`. |
| | `createdAt` | `Instant` | no | When `ConversationStarted` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the conversation reached a terminal state. |
| `MemoryStore` (entity state) | `sessionId` | `String` | no | Session this store belongs to. |
| | `items` | `List<MemoryItem>` | no | Append-only list; items may be superseded via `toForget`. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |

## Enums

- `ToolKind` → `CALCULATOR`, `KNOWLEDGE_BASE`, `WEB_LOOKUP`, `CODE_RUNNER`, `NONE`.
- `MemoryKind` → `FACT`, `PREFERENCE`, `ENTITY_MENTION`, `CORRECTION`.
- `TurnVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `ConversationStatus` → `IDLE`, `PROCESSING`, `HALTED`, `STUCK`, `ENDED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationStarted` | `conversationId, sessionId, createdAt` | → IDLE |
| `MessageReceived` | `turnNumber, text, receivedAt` | → PROCESSING |
| `TurnStarted` | `turnNumber, memoryItemCount` | no status change |
| `TurnBlocked` | `turnNumber, routingDecision, blockerReason` | no status change; appends TurnEntry with verdict BLOCKED_BY_GUARDRAIL |
| `TurnCompleted` | `entry: TurnEntry, newMemoryKeys: List<String>` | → IDLE; appends TurnEntry with verdict OK |
| `TurnFailed` | `turnNumber, errorReason` | → IDLE; appends TurnEntry with verdict FAILED |
| `ConversationHaltedOperator` | `haltReason, haltedAt` | → HALTED |
| `ConversationHaltedAutomatic` | `haltReason, haltedAt` | → HALTED |
| `ConversationFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`MemoryEntity`)

| Event | Payload |
|---|---|
| `MemoryInitialised` | `sessionId, initialisedAt` |
| `MemoryItemsAdded` | `items: List<MemoryItem>` (all values already scrubbed by PiiScrubber) |
| `MemoryItemsForgotten` | `keys: List<String>, forgottenAt` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`MessageQueue`)

| Event | Payload |
|---|---|
| `MessageSubmitted` | `conversationId, sessionId, text, sentBy, submittedAt` |

## View row

`ConversationRow` mirrors `Conversation` minus the full message history — `messageHistory` is truncated to the last 5 messages, and `turnLog.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count. Each entry's `reply` text is capped at 240 characters. The UI fetches the full conversation by id on click via `GET /api/conversations/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
