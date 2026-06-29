# Data model — telegram-tool-bot

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MessageRequest` | `text` | `String` | no | Inbound Telegram message text. |
| | `chatId` | `String` | no | Telegram chat identifier. |
| `SessionLedger` | `intent` | `String` | no | One-sentence summary of the user's goal. |
| | `facts` | `List<String>` | no | Facts the router believes are known. |
| | `toolPlan` | `List<String>` | no | Ordered list of tool steps (2–6). |
| | `currentDispatch` | `Optional<ToolDispatch>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `ToolDispatch` | `tool` | `ToolKind` | no | Which tool executor runs the subtask. |
| | `subtask` | `String` | no | One-sentence subtask description. |
| | `rationale` | `String` | no | One-sentence justification. |
| `ToolResult` | `tool` | `ToolKind` | no | Tool executor that ran the subtask. |
| | `subtask` | `String` | no | Echo of the subtask. |
| | `ok` | `boolean` | no | True if the executor could fulfil the subtask. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ToolEntry` | `attempt` | `int` | no | 1-based attempt count for this `(tool, subtask)` pair. |
| | `tool` | `ToolKind` | no | Tool executor that ran (or would have run). |
| | `subtask` | `String` | no | The subtask text. |
| | `verdict` | `ToolVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `scrubbedResult` | `String` | no | Sanitized result text. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ToolLedger` | `entries` | `List<ToolEntry>` | no | Append-only. |
| `BotReply` | `text` | `String` | no | 60–120 word reply text. |
| | `citations` | `List<String>` | no | 1–4 cited bullets, each tagged by tool. |
| | `producedAt` | `Instant` | no | When the router produced the reply. |
| `Session` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `chatId` | `String` | no | Telegram chat id. |
| | `text` | `String` | no | Original message text. |
| | `status` | `SessionStatus` | no | See enum. |
| | `ledger` | `Optional<SessionLedger>` | yes | Populated after `SessionPlanned`. |
| | `toolLedger` | `Optional<ToolLedger>` | yes | Populated after first `ToolCallRecorded` or `ToolCallBlocked`. |
| | `reply` | `Optional<BotReply>` | yes | Populated after `SessionCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `SessionFailed` / `SessionFailedTimeout`. |
| | `pauseReason` | `Optional<String>` | yes | Populated on `SessionPausedOperator`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |
| `BotControl` (entity state) | `paused` | `boolean` | no | Operator pause flag. |
| | `reason` | `Optional<String>` | yes | Set when `PauseRequested`. |
| | `pausedAt` | `Optional<Instant>` | yes | Set when `PauseRequested`; cleared when `PauseCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(ToolDispatch)`, `Replan(SessionLedger revised)`, `Reply(BotReply stub)`, `Fail(String reason)`. |

## Enums

- `ToolKind` → `WEB`, `CONTACTS`, `CALENDAR`, `NOTES`.
- `ToolVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `SessionStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `PAUSED`, `STALE`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `sessionId, chatId, text, createdAt` | → PLANNING |
| `SessionPlanned` | `ledger` | → EXECUTING |
| `ToolDispatched` | `dispatch` | no status change; sets `ledger.currentDispatch`. |
| `ToolCallBlocked` | `attempt, dispatch, blocker` | no status change; appends a `ToolEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `ToolCallRecorded` | `entry: ToolEntry` | no status change; appends to `toolLedger.entries`. |
| `LedgerRevised` | `ledger: SessionLedger` | no status change; replaces `ledger`. |
| `SessionCompleted` | `reply` | → COMPLETED, `finishedAt = now` |
| `SessionFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `SessionPausedOperator` | `pauseReason` | → PAUSED, `finishedAt = now` |
| `SessionFailedTimeout` | `failureReason` | → STALE, `finishedAt = now` |

## Events (`BotControlEntity`)

| Event | Payload |
|---|---|
| `PauseRequested` | `reason, pausedAt` |
| `PauseCleared` | `clearedAt` |

## Events (`MessageQueue`)

| Event | Payload |
|---|---|
| `MessageReceived` | `sessionId, text, chatId, receivedAt` |

## View row

`SessionRow` mirrors `Session` minus the heavy tool ledger payload — `toolLedger.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedResult` is capped at 240 characters. The UI fetches the full session by id on click via `GET /api/sessions/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
