# Data model — email-triage-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AttachmentContent` | `filename` | `String` | no | Original filename as submitted. |
| | `textContent` | `String` | no | Plain-text extracted content of the attachment. |
| `EmailSubmission` | `emailId` | `String` | no | UUID minted by `TriageEndpoint`. |
| | `fromAddress` | `String` | no | Sender's email address as submitted (raw; may be PII). |
| | `subject` | `String` | no | Email subject line. |
| | `rawBody` | `String` | no | Pre-sanitization email body. Audit-only. |
| | `attachments` | `List<AttachmentContent>` | no | Zero or more attachments (raw; audit-only). |
| | `submittedBy` | `String` | no | Operator identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedEmail` | `redactedBody` | `String` | no | PII stripped; this is what the agent sees as `body.txt`. |
| | `redactedAttachmentTexts` | `List<String>` | no | One redacted text per attachment; passed as `attachments.txt`. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","person-name","account-id","payment-card-number"]`. |
| `TriageResult` | `urgency` | `Urgency` | no | Enum value. |
| | `category` | `Category` | no | Enum value. |
| | `classificationRationale` | `String` | no | 1–2 sentences. |
| | `draftReply` | `String` | no | Full draft reply text. |
| | `classifiedAt` | `Instant` | no | When the agent returned. |
| `ConfirmationDecision` | `emailId` | `String` | no | Matches the parent email. |
| | `decision` | `ConfirmationStatus` | no | Enum value. |
| | `decidedBy` | `String` | no | Operator who clicked Approve / Reject. |
| | `decidedAt` | `Instant` | no | When the decision was recorded. |
| `Email` (entity state) | `emailId` | `String` | no | — |
| | `submission` | `Optional<EmailSubmission>` | yes | Populated after `EmailSubmitted`. |
| | `sanitized` | `Optional<SanitizedEmail>` | yes | Populated after `EmailSanitized`. |
| | `triage` | `Optional<TriageResult>` | yes | Populated after `DraftReady`. |
| | `confirmation` | `Optional<ConfirmationDecision>` | yes | Populated after `ConfirmationReceived`. |
| | `status` | `EmailStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `EmailSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `ConfirmationState` (entity state) | `emailId` | `String` | no | — |
| | `decision` | `Optional<ConfirmationDecision>` | yes | Populated after `DecisionRecorded`. |

Every nullable field on `Email` and `ConfirmationState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Urgency`: `HIGH`, `MEDIUM`, `LOW`.
`Category`: `BILLING`, `SUPPORT`, `LEGAL`, `GENERAL`.
`ConfirmationStatus`: `APPROVED`, `REJECTED`.
`EmailStatus`: `SUBMITTED`, `SANITIZED`, `REVIEWING`, `DRAFT_READY`, `SENDING`, `SENT`, `REJECTED`, `FAILED`.

## Events (`EmailEntity`)

| Event | Payload | Transition |
|---|---|---|
| `EmailSubmitted` | `submission` | → SUBMITTED |
| `EmailSanitized` | `sanitized` | → SANITIZED |
| `TriageStarted` | — | → REVIEWING |
| `DraftReady` | `triage` | → DRAFT_READY |
| `ConfirmationReceived` | `confirmation` | DRAFT_READY → SENDING (APPROVED) or DRAFT_READY → REJECTED (REJECTED) |
| `EmailSent` | `sentAt: Instant` | → SENT (terminal happy) |
| `EmailRejected` | `reason: String` | → REJECTED (terminal) |
| `EmailFailed` | `reason: String` | → FAILED (terminal) |

## Events (`ConfirmationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DecisionRecorded` | `decision` | → decision present |

`emptyState()` on both entities returns an initial-state record with all `Optional` fields as `Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`EmailRow` mirrors `Email` minus `submission.rawBody` and `submission.attachments[*].textContent` (the audit log keeps those). The UI fetches the raw data on demand via `GET /api/emails/{id}` and reads from `submission`.

The view declares ONE query: `getAllEmails: SELECT * AS emails FROM triage_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`EmailTasks.java`)

```java
public final class EmailTasks {
  public static final Task<TriageResult> TRIAGE_EMAIL = Task
      .name("Triage email")
      .description("Read the sanitized email body and attachments, classify urgency and category, draft a reply, and optionally call send_email when the user approves")
      .resultConformsTo(TriageResult.class);

  private EmailTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
