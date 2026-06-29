# Data model — meeting-to-tasks

Every record the generated system defines. Lifecycle fields nullable until their event fires are `Optional<T>` (Lesson 6).

## `Meeting` (event-sourced state and `meetings_view` row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Meeting id (UUID). |
| `title` | `String` | no | User-supplied title. |
| `rawNotes` | `String` | no | Notes as submitted, before redaction. |
| `status` | `MeetingStatus` | no | Lifecycle status enum. |
| `receivedAt` | `String` | no | ISO-8601 receipt time. |
| `sanitizedNotes` | `Optional<String>` | yes | Notes after PII redaction. |
| `redactionCount` | `Optional<Integer>` | yes | Number of redactions applied. |
| `extractedTasks` | `Optional<List<TaskItem>>` | yes | Task list from the agent. |
| `approvedAt` | `Optional<String>` | yes | Approval time. |
| `approvedBy` | `Optional<String>` | yes | Approver identifier. |
| `rejectedAt` | `Optional<String>` | yes | Rejection time. |
| `rejectReason` | `Optional<String>` | yes | Reason for rejection. |
| `trelloBoardUrl` | `Optional<String>` | yes | Board URL after push. |
| `trelloCardCount` | `Optional<Integer>` | yes | Cards created. |
| `csvPath` | `Optional<String>` | yes | Exported CSV path. |
| `slackMessageTs` | `Optional<String>` | yes | Chat message reference. |
| `completedAt` | `Optional<String>` | yes | Completion time. |
| `failedAt` | `Optional<String>` | yes | Failure time. |
| `failureReason` | `Optional<String>` | yes | Why the run failed. |

`emptyState()` returns a blank `Meeting` with placeholder id and `RECEIVED` status, with no `commandContext()` reference (Lesson 3).

## `TaskItem`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `title` | `String` | no | One-line action item. |
| `assignee` | `String` | no | Role word or redaction placeholder; never a real name. |
| `dueDate` | `String` | no | ISO date or empty string. |
| `priority` | `String` | no | `low` / `medium` / `high`. |

## `ExtractedTasks` (agent result)

| Field | Type | Meaning |
|---|---|---|
| `tasks` | `List<TaskItem>` | Extracted action items. |
| `summary` | `String` | One-sentence meeting outcome. |

## `ApprovalDecision`

| Field | Type | Meaning |
|---|---|---|
| `approvedBy` | `String` | Approver identifier. |

## `MeetingStatus` enum

`RECEIVED · SANITIZED · EXTRACTED · AWAITING_APPROVAL · APPROVED · REJECTED · COMPLETED · FAILED`

## Events (sealed `MeetingEvent`, each with `meetingId` and `timestamp`)

| Event | Trigger |
|---|---|
| `MeetingReceived` | `receive` command on intake. |
| `NotesSanitized` | `sanitizeStep` records redacted notes + count. |
| `TasksExtracted` | `extractStep` records the agent's task list. |
| `MeetingApproved` | `POST /approve`. |
| `MeetingRejected` | `POST /reject`. |
| `TasksPushedToBoard` | `pushToBoardStep` after the guardrail and TrelloClient call. |
| `CsvExported` | `exportCsvStep` after CsvWriter writes the file. |
| `SlackNotified` | `notifyStep` after the guardrail and SlackClient call. |
| `MeetingCompleted` | all three outputs done. |
| `MeetingFailed` | the workflow `error` recovery path. |

## View row

`MeetingsView` uses `Meeting` as its row type directly. One query: `getAllMeetings` (`SELECT * AS meetings FROM meetings_view`). No enum WHERE clause — status filtering happens client-side in the endpoint (Lesson 2).

## PII sanitizer (redaction rules)

`PiiSanitizer.redact(text)` replaces email addresses with `[EMAIL]` and attendee names with `[NAME]`, returning the redacted text and a count. The default name match keys off the attendee list at the top of each canned note plus a capitalized-pair heuristic; deployers tune these rules here.
