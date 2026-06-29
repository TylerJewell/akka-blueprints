# Data model — slack-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChannelContext` | `channelId` | `String` | no | Slack channel ID (e.g., `C01234ABCDE`). |
| | `channelName` | `String` | no | Human-readable channel name. |
| | `triggerUserId` | `String` | no | Slack user ID of the sender. |
| | `triggerText` | `String` | no | Raw message text. Audit-only after sanitization. |
| | `receivedAt` | `Instant` | no | When the endpoint received the submission. |
| `SanitizedContext` | `scrubbedText` | `String` | no | Secret-scrubbed message text; this is what the agent sees. |
| | `secretCategoriesFound` | `List<String>` | no | e.g. `["slack-token","gh-pat","aws-key-id"]`. Empty list if none found. |
| `AssistantReply` | `actionType` | `ActionType` | no | Enum value. |
| | `responseText` | `String` | no | 1–4 sentences, plain text. |
| | `runbookUrl` | `Optional<String>` | yes | Present only when `actionType == RUNBOOK_REF`. |
| | `guardrailIterations` | `int` | no | How many candidate responses the agent produced before one passed. |
| | `repliedAt` | `Instant` | no | When the agent returned. |
| `AuditRecord` | `candidateCount` | `int` | no | Total candidate responses produced by the agent. |
| | `rejectedCount` | `int` | no | How many were rejected by the guardrail. |
| | `outcome` | `String` | no | e.g. `"accepted-first-iteration"`, `"accepted-after-2-retries"`, `"failed-all-iterations"`. |
| | `auditedAt` | `Instant` | no | When `AuditRecorder` finished. |
| `Message` (entity state) | `messageId` | `String` | no | — |
| | `context` | `Optional<ChannelContext>` | yes | Populated after `MessageReceived`. |
| | `sanitized` | `Optional<SanitizedContext>` | yes | Populated after `MessageSanitized`. |
| | `reply` | `Optional<AssistantReply>` | yes | Populated after `ReplyRecorded`. |
| | `audit` | `Optional<AuditRecord>` | yes | Populated after `AuditCompleted`. |
| | `status` | `MessageStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Message` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ActionType`: `ANSWER`, `ESCALATE`, `RUNBOOK_REF`, `CLARIFY`.
`MessageStatus`: `RECEIVED`, `SANITIZED`, `REPLYING`, `REPLY_RECORDED`, `AUDITED`, `FAILED`.

## Events (`MessageEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MessageReceived` | `context` | → RECEIVED |
| `MessageSanitized` | `sanitized` | → SANITIZED |
| `ReplyStarted` | — | → REPLYING |
| `ReplyRecorded` | `reply` | → REPLY_RECORDED |
| `AuditCompleted` | `audit` | → AUDITED (terminal happy) |
| `MessageFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Message.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`MessageRow` mirrors `Message` minus `context.triggerText` raw form (the entity keeps that for audit). The UI fetches the raw text on demand via `GET /api/messages/{id}` and reads `context.triggerText` from the JSON.

The view declares ONE query: `getAllMessages: SELECT * AS messages FROM message_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`MessageTasks.java`)

```java
public final class MessageTasks {
  public static final Task<AssistantReply> REPLY_TO_MESSAGE = Task
      .name("Reply to message")
      .description("Read the attached sanitized message context and return an AssistantReply with an appropriate action type and response text")
      .resultConformsTo(AssistantReply.class);

  private MessageTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
