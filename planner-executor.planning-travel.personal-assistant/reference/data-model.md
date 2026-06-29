# Data model — line-personal-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundMessage` | `lineUserId` | `String` | no | LINE platform user identifier. |
| | `replyToken` | `String` | no | Single-use token for sending the LINE reply. |
| | `text` | `String` | no | Raw message text from the user. |
| | `receivedAt` | `Instant` | no | When the webhook was received. |
| `ActionPlan` | `intent` | `IntentKind` | no | Classified intent of the user's message. |
| | `reasoning` | `String` | no | One-sentence internal note (≤ 100 chars). |
| | `calendarParams` | `Optional<CalendarActionParams>` | yes | Populated when intent is `CREATE_EVENT` or `LIST_EVENTS`. |
| | `gmailParams` | `Optional<GmailActionParams>` | yes | Populated when intent is `SEND_EMAIL`. |
| | `clarificationText` | `Optional<String>` | yes | Populated when intent is `CLARIFY`. |
| `CalendarActionParams` | `action` | `CalendarAction` | no | `CREATE` or `LIST`. |
| | `title` | `String` | no | Event title (empty string for LIST). |
| | `description` | `Optional<String>` | yes | Optional event description. |
| | `startTime` | `Optional<Instant>` | yes | Required for `CREATE`; range start for `LIST`. |
| | `endTime` | `Optional<Instant>` | yes | Required for `CREATE`; range end for `LIST`. |
| | `timeZone` | `Optional<String>` | yes | IANA timezone string; defaults to UTC. |
| `GmailActionParams` | `recipientEmail` | `String` | no | To: address. |
| | `subject` | `String` | no | Email subject line (≤ 60 chars). |
| | `body` | `String` | no | Plain-text email body. |
| | `ccEmail` | `Optional<String>` | yes | Optional Cc: address. |
| `ExecutionResult` | `intent` | `IntentKind` | no | Which intent was executed. |
| | `ok` | `boolean` | no | True if the API call succeeded. |
| | `summary` | `String` | no | Human-readable outcome (used in LINE reply). |
| | `errorReason` | `Optional<String>` | yes | API error message when `ok = false`. |
| | `executedAt` | `Instant` | no | Execution timestamp. |
| `ConfirmationRequest` | `conversationId` | `String` | no | Links back to the `ConversationEntity`. |
| | `params` | `GmailActionParams` | no | The email params awaiting approval. |
| | `requestedAt` | `Instant` | no | When the confirmation was opened. |
| | `respondedAt` | `Optional<Instant>` | yes | When the user responded (or null if pending/expired). |
| | `status` | `ConfirmationStatus` | no | `PENDING`, `APPROVED`, `DENIED`, `EXPIRED`. |
| `Conversation` (entity state) | `conversationId` | `String` | no | Unique id. |
| | `lineUserId` | `String` | no | LINE user who sent the message. |
| | `originalText` | `String` | no | The user's raw message. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `plan` | `Optional<ActionPlan>` | yes | Populated after `PlanProduced`. |
| | `result` | `Optional<ExecutionResult>` | yes | Populated after `ActionExecuted`. |
| | `blockerReason` | `Optional<String>` | yes | Populated when `ActionBlocked`. |
| | `createdAt` | `Instant` | no | When `ConversationCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the conversation reached a terminal state. |

## Enums

- `IntentKind` → `CREATE_EVENT`, `LIST_EVENTS`, `SEND_EMAIL`, `CLARIFY`.
- `CalendarAction` → `CREATE`, `LIST`.
- `ConversationStatus` → `PLANNING`, `AWAITING_CONFIRMATION`, `AWAITING_CLARIFICATION`, `EXECUTING`, `COMPLETED`, `CANCELLED`, `BLOCKED`, `EXPIRED`, `FAILED`.
- `ConfirmationStatus` → `PENDING`, `APPROVED`, `DENIED`, `EXPIRED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationCreated` | `conversationId, lineUserId, originalText, createdAt` | → PLANNING |
| `PlanProduced` | `plan: ActionPlan` | → EXECUTING (calendar intents); no status change on SEND_EMAIL (goes to ConfirmationRequested next) |
| `ActionBlocked` | `blockerReason` | → BLOCKED, `finishedAt = now` |
| `ConfirmationRequested` | `params: GmailActionParams` | → AWAITING_CONFIRMATION |
| `ConfirmationApproved` | `respondedAt` | → EXECUTING |
| `ConfirmationDenied` | `respondedAt` | → CANCELLED, `finishedAt = now` |
| `ActionExecuted` | `result: ExecutionResult` | → COMPLETED, `finishedAt = now` |
| `ClarificationRequested` | `clarificationText` | → AWAITING_CLARIFICATION |
| `ActionCancelled` | `reason` | → CANCELLED, `finishedAt = now` |
| `ConversationExpired` | `expiredAt` | → EXPIRED, `finishedAt = now` |
| `ConversationFailed` | `failureReason` | → FAILED, `finishedAt = now` |

## Events (`ConfirmationEntity`)

| Event | Payload |
|---|---|
| `ConfirmationOpened` | `conversationId, params, requestedAt` |
| `ConfirmationResolved` | `approved: boolean, respondedAt` |
| `ConfirmationTimedOut` | `timedOutAt` |

## Events (`MessageQueue`)

| Event | Payload |
|---|---|
| `MessageEnqueued` | `conversationId, lineUserId, text, receivedAt` |

## View row

`ConversationRow` mirrors `Conversation` with `plan.gmailParams.body` truncated to 120 chars and `result.summary` truncated to 120 chars. The UI fetches the full conversation by id on expand via `GET /api/conversations/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
