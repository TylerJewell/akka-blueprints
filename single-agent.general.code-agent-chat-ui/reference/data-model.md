# Data model — code-agent-chat-ui

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChatMessage` | `messageId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `role` | `MessageRole` | no | `USER`, `ASSISTANT`, or `SYSTEM`. |
| | `content` | `String` | no | Message body (markdown for ASSISTANT turns). |
| | `sentAt` | `Instant` | no | When the message was recorded. |
| | `toolCalls` | `Optional<List<ToolCallRecord>>` | yes | Non-null only on ASSISTANT turns that made tool calls. |
| `ToolCallRecord` | `toolCallId` | `String` | no | UUID minted at request time. |
| | `toolName` | `String` | no | `"web-search"` or `"code-execution"`. |
| | `inputSummary` | `String` | no | Truncated to 200 chars for display. |
| | `status` | `ToolCallStatus` | no | Enum value. |
| | `blockReason` | `Optional<String>` | yes | Non-null when `status = BLOCKED`. |
| | `outputSummary` | `Optional<String>` | yes | Non-null when `status = COMPLETED`. |
| | `requestedAt` | `Instant` | no | When the tool call was requested. |
| | `resolvedAt` | `Optional<Instant>` | yes | Non-null when resolved (approved, blocked, or completed). |
| `PlanRevision` | `stepNumberAtRevision` | `int` | no | Step count at which the revision triggered (always a multiple of 3). |
| | `revisedGoal` | `String` | no | Agent's stated goal after re-planning. |
| | `revisedAt` | `Instant` | no | When the revision was emitted. |
| `ChatResponse` | `responseText` | `String` | no | Markdown; validated by `ResponseGuardrail`. |
| | `toolCallLog` | `List<ToolCallRecord>` | no | All tool calls from this turn, in order. |
| | `planRevisions` | `List<PlanRevision>` | no | Any plan revisions emitted during this turn. |
| | `generatedAt` | `Instant` | no | When the agent returned. |
| `ChatSession` | `sessionId` | `String` | no | — |
| | `title` | `String` | no | Auto-derived from first user message; `"Untitled session"` until first message. |
| | `messages` | `List<ChatMessage>` | no | All turns in order; empty on create. |
| | `planRevisions` | `List<PlanRevision>` | no | All plan revisions across all turns; empty on create. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `lastActiveAt` | `Optional<Instant>` | yes | Updated on every `UserMessageReceived` and `AssistantMessageRecorded`. |

Every nullable field on `ChatSession` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`MessageRole`: `USER`, `ASSISTANT`, `SYSTEM`.
`ToolCallStatus`: `PENDING`, `APPROVED`, `BLOCKED`, `COMPLETED`, `FAILED`.
`SessionStatus`: `INITIALIZING`, `ACTIVE`, `IDLE`, `FAILED`.

## Events (`ChatSessionEntity`)

| Event | Payload | Trigger |
|---|---|---|
| `SessionCreated` | `title: String` | → INITIALIZING |
| `UserMessageReceived` | `message: ChatMessage` | stays in current state; signals workflow |
| `AssistantMessageRecorded` | `message: ChatMessage` | stays in ACTIVE; workflow advances |
| `ToolCallRequested` | `record: ToolCallRecord (status=PENDING)` | emitted before tool runs |
| `ToolCallResolved` | `toolCallId, status, outputSummary, blockReason` | emitted after `ToolCallValidator` processes |
| `PlanRevised` | `revision: PlanRevision` | at every 3-step boundary |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ChatSession.initial("")` with `messages = []`, `planRevisions = []`, `status = INITIALIZING`, and all `Optional` fields as `Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ChatSessionRow` mirrors `ChatSession` minus the per-message `content` (full content available via `GET /api/sessions/{id}`). Carries `messageCount` (derived in the table updater), `status`, `title`, `createdAt`, and `lastActiveAt`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM chat_session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AgentTasks.java`)

```java
public final class AgentTasks {
  public static final Task<ChatResponse> ANSWER_QUESTION = Task
      .name("Answer question")
      .description("Use available tools to answer the user's question and return a ChatResponse with markdown responseText")
      .resultConformsTo(ChatResponse.class);

  private AgentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
