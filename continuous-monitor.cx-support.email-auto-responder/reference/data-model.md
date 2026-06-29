# Data model

## `Email` (event-sourced state and `EmailsView` row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Email id (UUID). |
| `fromAddress` | `String` | no | Sender address, present at ingest. |
| `subject` | `String` | no | Original subject, present at ingest. |
| `body` | `String` | no | Raw body, present at ingest. |
| `status` | `EmailStatus` | no | Lifecycle state. |
| `receivedAt` | `Optional<Instant>` | yes | Set on ingest. |
| `sanitizedBody` | `Optional<String>` | yes | Body with PII masked; set by `sanitizeStep`. |
| `draftSubject` | `Optional<String>` | yes | Drafted reply subject. |
| `draftBody` | `Optional<String>` | yes | Drafted reply body. |
| `sensitive` | `Optional<Boolean>` | yes | Triage result; null until drafted. |
| `draftedAt` | `Optional<Instant>` | yes | Set when the reply is drafted. |
| `approvedAt` | `Optional<Instant>` | yes | Set on approve. |
| `approvedBy` | `Optional<String>` | yes | Approver id. |
| `rejectedAt` | `Optional<Instant>` | yes | Set on reject. |
| `rejectedBy` | `Optional<String>` | yes | Rejecter id. |
| `rejectReason` | `Optional<String>` | yes | Reason given on reject. |
| `sentAt` | `Optional<Instant>` | yes | Set when the reply is sent. |
| `sentMessageId` | `Optional<String>` | yes | Simulated send message id. |
| `escalatedAt` | `Optional<Instant>` | yes | Set on escalation. |

Every nullable lifecycle field is `Optional<T>` (Lesson 6). `Email.initial(id)` returns all `Optional.empty()` with `status = RECEIVED`. `emptyState()` returns `Email.initial("")` and never calls `commandContext()` (Lesson 3).

## `EmailStatus` (enum)

`RECEIVED · SANITIZED · DRAFTED · AWAITING_REVIEW · APPROVED · REJECTED · SENT · ESCALATED`

## `EmailEvent` (sealed interface)

```java
sealed interface EmailEvent permits EmailIngested, EmailSanitized, ReplyDrafted,
                                     ReplyApproved, ReplyRejected, ReplySent,
                                     EmailEscalated {
  Instant timestamp();
}
```

| Event | Trigger |
|---|---|
| `EmailIngested(id, fromAddress, subject, body, timestamp)` | `InboundEmailConsumer.ingest` on a new email. |
| `EmailSanitized(id, sanitizedBody, timestamp)` | `ResponderWorkflow.sanitizeStep`. |
| `ReplyDrafted(id, draftSubject, draftBody, sensitive, timestamp)` | `ResponderWorkflow.draftStep` after triage + responder. |
| `ReplyApproved(id, approvedBy, timestamp)` | POST `/api/emails/{id}/approve`. |
| `ReplyRejected(id, rejectedBy, reason, timestamp)` | POST `/api/emails/{id}/reject`. |
| `ReplySent(id, sentMessageId, timestamp)` | `ResponderWorkflow.sendStep`. |
| `EmailEscalated(id, timestamp)` | `StuckReviewMonitor` past the review threshold. |

## `InboxQueue` events

`EmailQueued(EmailIn email, Instant timestamp)` — emitted by `InboxQueue.receive`.

## Auxiliary records

```java
record EmailIn(String fromAddress, String subject, String body) {}
record DraftReply(String subject, String body) {}
record Triage(boolean sensitive, String reason) {}
```

## View row type

`EmailsView` uses `Email` as its row type directly. One query: `getAllEmails` → `SELECT * AS emails FROM emails_view`. No enum `WHERE` clause (Lesson 2); callers filter by status client-side.
