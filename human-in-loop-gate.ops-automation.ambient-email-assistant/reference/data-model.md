# Data model

Every record, event, enum, and view row the generated system defines.

## `EmailThread` (EmailThreadEntity state + ThreadsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Thread id; equals the workflow id and entity id |
| `sender` | `Optional<String>` | yes | Sender address from the submitted email |
| `subject` | `Optional<String>` | yes | Subject line from the submitted email |
| `status` | `ThreadStatus` | no | Lifecycle status |
| `receivedAt` | `Optional<Instant>` | yes | When the thread was first received |
| `category` | `Optional<String>` | yes | Triage classification category |
| `urgency` | `Optional<String>` | yes | Triage urgency level |
| `suggestedAction` | `Optional<String>` | yes | Triage-suggested action (`reply`, `meeting`, `ignore`) |
| `triageScore` | `Optional<Double>` | yes | LLM-as-judge confidence score from E1 eval event |
| `draftSubject` | `Optional<String>` | yes | Reply draft subject line |
| `draftBody` | `Optional<String>` | yes | Reply draft body |
| `proposedMeetingTitle` | `Optional<String>` | yes | Meeting proposal title |
| `proposedMeetingTime` | `Optional<String>` | yes | ISO-8601 proposed meeting start time |
| `proposedAttendees` | `Optional<String>` | yes | Comma-separated attendee identifiers |
| `approvedAt` | `Optional<Instant>` | yes | When the operator approved the action |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverNote` | `Optional<String>` | yes | Approver note |
| `dismissedAt` | `Optional<Instant>` | yes | When the operator dismissed the thread |
| `dismissedBy` | `Optional<String>` | yes | Dismisser identity |
| `dismissNote` | `Optional<String>` | yes | Dismiss reason |
| `actionCompletedAt` | `Optional<Instant>` | yes | When the outbound action completed |
| `actionReference` | `Optional<String>` | yes | External action id (e.g., `MSG-<uuid>` or `EVT-<uuid>`) |

Every nullable lifecycle field is `Optional<T>` because `EmailThread` is the view row type (Lesson 6). `emptyState()` returns `EmailThread.initial("")` with no `commandContext()` reference (Lesson 3).

## `ThreadStatus` enum

`RECEIVED`, `TRIAGED`, `ACTION_DRAFTED`, `APPROVED`, `REPLY_SENT`, `MEETING_SCHEDULED`, `DISMISSED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `ThreadReceived` | `recordReceived` when EmailWorkflow starts | sets `receivedAt`, `sender`, `subject`, status → `RECEIVED` |
| `ThreadTriaged` | `recordTriage` after TriageAgent returns | sets `category`, `urgency`, `suggestedAction`, `triageScore`, status → `TRIAGED` |
| `ActionDrafted` | `recordDraft` after ReplyDraftAgent or MeetingSchedulerAgent returns | sets draft fields or meeting fields, status → `ACTION_DRAFTED` |
| `ThreadApproved` | `approve` command (operator via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `ThreadDismissed` | `dismiss` command (operator via API) | sets `dismissedAt`, `dismissedBy`, `dismissNote`, status → `DISMISSED` |
| `ReplySent` | `recordActionComplete` after Gmail send tool returns | sets `actionCompletedAt`, `actionReference`, status → `REPLY_SENT` |
| `MeetingScheduled` | `recordActionComplete` after Calendar create tool returns | sets `actionCompletedAt`, `actionReference`, status → `MEETING_SCHEDULED` |

## Domain records

- `EmailClassification(String category, String urgency, String suggestedAction)` — TriageAgent result.
- `ReplyDraft(String subject, String body)` — ReplyDraftAgent result.
- `MeetingProposal(String title, String proposedTime, String attendees)` — MeetingSchedulerAgent result.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `DismissDecision(String dismissedBy, String note)` — dismiss payload.
- `EvalEvent(String threadId, String agentName, String inputSummary, String outputSummary, double judgeScore, Instant evaluatedAt)` — emitted after each triage and draft decision.

## View row type

`ThreadsView` rows are `EmailThread`. One query: `getAllThreads` → `SELECT * AS threads FROM threads_view`. No `WHERE status` filter; callers filter by status or category client-side (Lesson 2).

## Task constants (EmailTasks.java)

- `TRIAGE` — `Task.name(...).resultConformsTo(EmailClassification.class)`.
- `REPLY_DRAFT` — `Task.name(...).resultConformsTo(ReplyDraft.class)`.
- `MEETING` — `Task.name(...).resultConformsTo(MeetingProposal.class)`.
