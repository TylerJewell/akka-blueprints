# Data model

Every nullable lifecycle field is `Optional<T>` so the View materializer accepts the row (Lesson 6). `actions` is a non-null `List<ActionItem>` defaulting to empty.

## `Meeting` (event-sourced state and View row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Meeting id (UUID). |
| `title` | `Optional<String>` | yes | User-supplied title. |
| `status` | `MeetingStatus` | no | Lifecycle stage. |
| `receivedAt` | `Optional<Instant>` | yes | When notes were received. |
| `sanitizedNotes` | `Optional<String>` | yes | Redacted transcript. |
| `redactionCount` | `Optional<Integer>` | yes | Number of redactions applied. |
| `sanitizedAt` | `Optional<Instant>` | yes | When redaction finished. |
| `summary` | `Optional<String>` | yes | Agent summary. |
| `actions` | `List<ActionItem>` | no (empty default) | Extracted action items. |
| `extractedAt` | `Optional<Instant>` | yes | When extraction finished. |
| `dispatchedAt` | `Optional<Instant>` | yes | When fan-out finished. |
| `trelloBoardId` | `Optional<String>` | yes | Target Trello board. |
| `slackChannel` | `Optional<String>` | yes | Target Slack channel. |
| `slackMessageTs` | `Optional<String>` | yes | Slack message timestamp. |
| `failureReason` | `Optional<String>` | yes | Guardrail/dispatch failure reason. |
| `failedAt` | `Optional<Instant>` | yes | When failure was recorded. |

```java
public static Meeting initial(String id) { /* status RECEIVED, all Optional.empty(), actions = List.of() */ }
public Meeting applyEvent(MeetingEvent e) { /* per-variant switch */ }
```

## `ActionItem`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `title` | `String` | no | Imperative action title. |
| `assignee` | `Optional<String>` | yes | Role or redaction token if named. |
| `dueHint` | `Optional<String>` | yes | Date or relative phrase if stated. |
| `trelloCardUrl` | `Optional<String>` | yes | Filled after the card is created. |

## `ActionPlan` (agent result)

`record ActionPlan(String summary, List<ActionItem> actions)`.

## `MeetingStatus` (enum)

`RECEIVED · SANITIZED · EXTRACTED · DISPATCHED · FAILED`

## Events (`MeetingEvent` sealed interface)

| Event | Trigger |
|---|---|
| `NotesReceived(meetingId, title, receivedAt)` | `MeetingEntity.receive` after submission. |
| `NotesSanitized(meetingId, sanitizedNotes, redactionCount, timestamp)` | `sanitizeStep` after `PiiRedactor` runs. |
| `ActionsExtracted(meetingId, summary, actions, timestamp)` | `extractStep` after the agent returns. |
| `ActionsDispatched(meetingId, actions, slackMessageTs, timestamp)` | `dispatchStep` after all writes succeed. |
| `DispatchFailed(meetingId, reason, timestamp)` | `dispatchStep` on guardrail refusal or write failure. |

```java
sealed interface MeetingEvent permits NotesReceived, NotesSanitized,
    ActionsExtracted, ActionsDispatched, DispatchFailed {
  Instant timestamp();
}
```

## `InboundNotesQueue`

State holds the latest queued submission; one event `InboundNotesQueued(meetingId, title, rawNotes, timestamp)` emitted by `enqueueNotes`. `NotesConsumer` subscribes and starts a `PipelineWorkflow`.

## View row

`MeetingsView` row type is `Meeting`. One query `getAllMeetings` (`SELECT * AS meetings FROM meetings_view`) plus a `streamAllMeetings` method for SSE. No `WHERE status` clause — status filtering is client-side (Lesson 2).
