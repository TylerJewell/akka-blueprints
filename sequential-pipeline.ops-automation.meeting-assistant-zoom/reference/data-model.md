# Data model — meeting-assistant-zoom

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Segment` | `speaker` | `String` | no | Pseudonym from `PiiSanitizer` output (e.g. `[PERSON-1]`). |
| | `text` | `String` | no | Sanitized text of this speaker turn. |
| | `startTime` | `Instant` | no | Wall-clock start of this turn. |
| | `endTime` | `Instant` | no | Wall-clock end of this turn. |
| `RedactionEntry` | `original` | `String` | no | The original PII string (stored only in the audit; never forwarded to the LLM). |
| | `pseudonym` | `String` | no | The stable pseudonym replacing it. |
| | `patternType` | `String` | no | `name`, `email`, or `phone`. |
| `RedactionAudit` | `totalRedactions` | `int` | no | Count of distinct PII occurrences replaced. |
| | `entries` | `List<RedactionEntry>` | no | One entry per distinct original value. May be empty if no PII was found. |
| `Transcript` | `segments` | `List<Segment>` | no | Possibly empty for a fully silent recording. |
| | `redactionAudit` | `RedactionAudit` | no | Always present; may show zero redactions. |
| | `capturedAt` | `Instant` | no | When the TRANSCRIBE task returned. |
| `ActionItem` | `actionId` | `String` | no | Short stable id (`ai-<8 hex>`). |
| | `description` | `String` | no | What was agreed. |
| | `assigneePseudonym` | `String` | no | MUST equal a `Segment.speaker` from the upstream `Transcript`. |
| | `dueDate` | `Optional<LocalDate>` | yes | Only present if a due date was mentioned. |
| `Decision` | `decisionId` | `String` | no | Short stable id (`d-<8 hex>`). |
| | `text` | `String` | no | The agreed position. |
| | `decidedAt` | `Instant` | no | Timestamp of the segment where the decision was stated. |
| `MeetingSummary` | `summaryText` | `String` | no | 2–3 sentence summary. |
| | `actionItems` | `List<ActionItem>` | no | Possibly empty for informational meetings. |
| | `decisions` | `List<Decision>` | no | Possibly empty. |
| | `summarizedAt` | `Instant` | no | When the SUMMARIZE task returned. |
| `FollowUpEvent` | `eventId` | `String` | no | Short stable id. |
| | `title` | `String` | no | Calendar event title. |
| | `attendeePseudonyms` | `List<String>` | no | Every entry MUST be a `Segment.speaker` from the `Transcript`. |
| | `scheduledDate` | `LocalDate` | no | Proposed date. |
| `TaskRecord` | `taskId` | `String` | no | Short stable id. |
| | `assigneePseudonym` | `String` | no | MUST be a `Segment.speaker` from the `Transcript`. |
| | `description` | `String` | no | Task description. |
| | `dueDate` | `LocalDate` | no | Due date. |
| `MeetingPackage` | `title` | `String` | no | Meeting title + "— Follow-ups". |
| | `summaryText` | `String` | no | Mirrors `MeetingSummary.summaryText`. |
| | `actionItems` | `List<ActionItem>` | no | Mirrors `MeetingSummary.actionItems`. |
| | `followUpEvents` | `List<FollowUpEvent>` | no | Possibly empty. |
| | `tasks` | `List<TaskRecord>` | no | One per `ActionItem`. |
| | `packagedAt` | `Instant` | no | When the DISPATCH task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `ActionItemScorer` finished. |
| `ScopeRejection` | `phase` | `String` | no | `TRANSCRIBE` / `SUMMARIZE` / `DISPATCH`. |
| | `tool` | `String` | no | Name of the rejected tool. |
| | `reason` | `String` | no | Structured reason from `ScopeGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `MeetingRecord` (entity state) | `meetingId` | `String` | no | — |
| | `meetingTitle` | `Optional<String>` | yes | Populated after `MeetingCreated`. |
| | `transcript` | `Optional<Transcript>` | yes | Populated after `TranscriptProduced`. |
| | `summary` | `Optional<MeetingSummary>` | yes | Populated after `SummaryProduced`. |
| | `meetingPackage` | `Optional<MeetingPackage>` | yes | Populated after `FollowUpsDispatched`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `MeetingStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MeetingCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `scopeRejections` | `List<ScopeRejection>` | no | Appended on every `ScopeRejected` event; empty on the happy path. |

Every nullable lifecycle field on `MeetingRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`MeetingStatus`: `CREATED`, `TRANSCRIBING`, `TRANSCRIBED`, `SUMMARIZING`, `SUMMARIZED`, `DISPATCHING`, `DISPATCHED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `ScopeGuardrail`): `TRANSCRIBE`, `SUMMARIZE`, `DISPATCH`.

## Events (`MeetingEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MeetingCreated` | `meetingTitle: String, rawTranscriptEncrypted: byte[]` | → CREATED |
| `TranscribeStarted` | `redactionAudit: RedactionAudit` | → TRANSCRIBING |
| `TranscriptProduced` | `transcript: Transcript` | → TRANSCRIBED |
| `SummarizeStarted` | — | → SUMMARIZING |
| `SummaryProduced` | `summary: MeetingSummary` | → SUMMARIZED |
| `DispatchStarted` | — | → DISPATCHING |
| `FollowUpsDispatched` | `meetingPackage: MeetingPackage` | → DISPATCHED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `ScopeRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `MeetingFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `MeetingRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `scopeRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`MeetingRow` mirrors `MeetingRecord` exactly. The UI fetches the full row via `GET /api/meetings/{id}` and streams updates via `GET /api/meetings/sse`.

The view declares ONE query: `getAllMeetings: SELECT * AS meetings FROM meeting_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`MeetingTasks.java`)

```java
public final class MeetingTasks {
  public static final Task<Transcript> TRANSCRIBE_MEETING = Task
      .name("Transcribe meeting")
      .description("Segment the sanitized transcript and label each speaker turn")
      .resultConformsTo(Transcript.class);

  public static final Task<MeetingSummary> SUMMARIZE_MEETING = Task
      .name("Summarize meeting")
      .description("Extract action items, decisions, and a summary paragraph from the transcript")
      .resultConformsTo(MeetingSummary.class);

  public static final Task<MeetingPackage> DISPATCH_FOLLOWUPS = Task
      .name("Dispatch follow-ups")
      .description("Schedule follow-up calendar events and create tasks from the action items")
      .resultConformsTo(MeetingPackage.class);

  private MeetingTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `TranscribeTools`, `SummarizeTools`, and `DispatchTools` carries a `Phase` constant. `ScopeGuardrail` reads this constant before the tool body runs and applies the per-status accept matrix (phase-order check) plus, for `DispatchTools` methods, the participant-list check (write-scope check). The tool registry is built once at startup; the guardrail reads it for every call.
