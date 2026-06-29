# Data model — ae-oauth

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TokenSpec` | `tokenId` | `String` | no | Stable token identifier, looked up in `TokenRegistry`. |
| | `scopes` | `List<String>` | no | Full scope set granted to this token. |
| | `issuedAt` | `Instant` | no | When the token was issued. |
| | `expiresAt` | `Instant` | no | When the token expires. Checked in `resolveTokenStep`. |
| `SessionRequest` | `sessionId` | `String` | no | UUID minted by `SessionEndpoint`. |
| | `tokenId` | `String` | no | Opaque token id submitted by the caller. |
| | `requestText` | `String` | no | Natural-language request from the caller. |
| | `submittedBy` | `String` | no | Caller identifier (application or user). |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ResolvedToken` | `tokenId` | `String` | no | Same as submitted; included for traceability. |
| | `scopes` | `List<String>` | no | Scope set confirmed by `TokenRegistry`. |
| | `expired` | `boolean` | no | `true` if `expiresAt` was in the past at resolve time. |
| `ToolCallRecord` | `toolName` | `String` | no | Name of the tool proposed by the agent. |
| | `requiredScope` | `String` | no | Scope this tool requires, from `ToolRegistry`. |
| | `disposition` | `Disposition` | no | Enum: ALLOWED or DENIED. |
| | `output` | `String` | no | Tool output (ALLOWED) or denial reason (DENIED). |
| | `calledAt` | `Instant` | no | When the guardrail evaluated this call. |
| `AgentResult` | `outcome` | `Outcome` | no | Enum: SUCCESS, PARTIAL, or DENIED. |
| | `summary` | `String` | no | 1–3 sentences describing what happened. |
| | `toolCalls` | `List<ToolCallRecord>` | no | One entry per attempted tool call, in proposal order. |
| | `completedAt` | `Instant` | no | When `ToolCallerAgent` returned. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `request` | `Optional<SessionRequest>` | yes | Populated after `SessionCreated`. |
| | `token` | `Optional<ResolvedToken>` | yes | Populated after `TokenResolved`. |
| | `result` | `Optional<AgentResult>` | yes | Populated after `ResultRecorded`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Session` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Disposition`: `ALLOWED`, `DENIED`.
`Outcome`: `SUCCESS`, `PARTIAL`, `DENIED`.
`SessionStatus`: `PENDING`, `TOKEN_RESOLVED`, `RUNNING`, `COMPLETED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `request` | → PENDING |
| `TokenResolved` | `token` | → TOKEN_RESOLVED |
| `AgentStarted` | — | → RUNNING |
| `ResultRecorded` | `result` | → COMPLETED |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Session.initial("")` with all `Optional` fields as `Optional.empty()` and `status = PENDING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session`. The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SessionTasks.java`)

```java
public final class SessionTasks {
  public static final Task<AgentResult> CALL_TOOLS = Task
      .name("Call tools")
      .description("Parse the request, check OAuth scopes via the guardrail on each proposed call, and return an AgentResult with one ToolCallRecord per attempted call")
      .resultConformsTo(AgentResult.class);

  private SessionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Tool registry (static, in-process)

| Tool name | Required scope |
|---|---|
| `listCalendarEvents` | `calendar:read` |
| `createCalendarEvent` | `calendar:write` |
| `listFiles` | `files:read` |
| `uploadFile` | `files:write` |
| `listContacts` | `contacts:read` |

## Token registry (seeded fixtures)

| Token id | Scopes | Expires |
|---|---|---|
| `full-access` | `calendar:read`, `calendar:write`, `files:read`, `files:write`, `contacts:read` | 2099-01-01 |
| `read-only` | `calendar:read`, `files:read`, `contacts:read` | 2099-01-01 |
| `calendar-only` | `calendar:read`, `calendar:write` | 2099-01-01 |
| `expired-token` | `calendar:read` | 2020-01-01 |
