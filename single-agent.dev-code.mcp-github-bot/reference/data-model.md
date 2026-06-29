# Data model — mcp-github-bot

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BotRequest` | `sessionId` | `String` | no | UUID minted by `BotSessionEndpoint`. |
| | `userRequest` | `String` | no | Natural-language request text. |
| | `repository` | `String` | no | Target in `owner/repo` format. |
| | `githubToken` | `String` | no | MCP auth credential. **Never persisted in entity state or view.** |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallRecord` | `toolName` | `String` | no | e.g. `list_issues`. |
| | `inputSummary` | `String` | no | JSON-serialized inputs, max 500 chars. |
| | `outcome` | `String` | no | `"success"`, `"error"`, or `"blocked"`. |
| | `resultSummary` | `String` | no | First 500 chars of result or rejection reason. |
| | `calledAt` | `Instant` | no | When the tool dispatch was attempted. |
| `BotResponse` | `agentMessage` | `String` | no | 1–3-sentence natural-language summary. |
| | `toolCalls` | `List<ToolCallRecord>` | no | Ordered list of all tool calls attempted. |
| | `blockedCallCount` | `int` | no | Count of guardrail-blocked calls. |
| | `respondedAt` | `Instant` | no | When `GitHubBotAgent` returned. |
| `BotSession` (entity state) | `sessionId` | `String` | no | — |
| | `request` | `Optional<BotRequest>` | yes | Populated after `SessionSubmitted` (without `githubToken`). |
| | `response` | `Optional<BotResponse>` | yes | Populated after `SessionCompleted`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `HaltFlag` (entity state) | `writeHalted` | `boolean` | no | True when write operations are blocked. |
| | `haltedBy` | `Optional<String>` | yes | Operator who last enabled the halt. |
| | `haltedAt` | `Optional<Instant>` | yes | When the halt was last enabled. |

Every nullable field on `BotSession` and `HaltFlag` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

`BotRequest.githubToken` is used only as a workflow-start parameter forwarded to `McpServerConfig`. It must not appear in `SessionSubmitted` event payload, `BotSession` state, `BotSessionRow` view row, or any HTTP response body.

## Enums

`SessionStatus`: `SUBMITTED`, `RUNNING`, `COMPLETED`, `FAILED`.

## Events

### `BotSessionEntity` events

| Event | Payload | Transition |
|---|---|---|
| `SessionSubmitted` | `userRequest`, `repository`, `submittedBy`, `submittedAt` (no `githubToken`) | → SUBMITTED |
| `SessionStarted` | — | → RUNNING |
| `SessionCompleted` | `response: BotResponse` | → COMPLETED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `BotSession.initial("")` with all `Optional` fields as `Optional.empty()`, `status = SUBMITTED`, `createdAt = Instant.EPOCH`. `emptyState()` never references `commandContext()` (Lesson 3).

### `HaltFlagEntity` events

| Event | Payload | Effect |
|---|---|---|
| `WriteHaltEnabled` | `enabledBy: String`, `enabledAt: Instant` | `writeHalted = true` |
| `WriteHaltDisabled` | `disabledBy: String`, `disabledAt: Instant` | `writeHalted = false` |

`emptyState()` returns `HaltFlag{writeHalted: false, haltedBy: Optional.empty(), haltedAt: Optional.empty()}`.

## View row

`BotSessionRow` mirrors `BotSession` but flattens `response` into top-level convenience fields: `agentMessage: Optional<String>` and `blockedCallCount: Optional<Integer>`. The full `toolCalls` list is included for the detail pane. The `githubToken` field is absent entirely.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM bot_session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`BotTasks.java`)

```java
public final class BotTasks {
  public static final Task<BotResponse> EXECUTE_GITHUB_REQUEST = Task
      .name("Execute GitHub request")
      .description("Translate the natural-language request into GitHub MCP tool calls and return a BotResponse")
      .resultConformsTo(BotResponse.class);

  private BotTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
