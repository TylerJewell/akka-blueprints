# Data model — meeting-preparer

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CrmSnapshot` | `contactName` | `String` | no | Raw PII — contact's full name. Audit-only after sanitization. |
| | `contactEmail` | `String` | no | Raw PII — contact's email address. |
| | `contactPhone` | `String` | no | Raw PII — contact's phone number. |
| | `dealStage` | `String` | no | Current relationship stage from CRM (e.g., "Active — renewal"). |
| | `lastInteractionNote` | `String` | no | Free-text note from the last CRM interaction. May contain identifiers. |
| | `lastContactAt` | `Instant` | no | Timestamp of last logged CRM interaction. |
| `MeetingRequest` | `briefId` | `String` | no | UUID minted by `BriefingEndpoint`. |
| | `counterpartyName` | `String` | no | Name of the firm or individual the meeting is with. |
| | `meetingAt` | `Instant` | no | Scheduled meeting time. |
| | `attendees` | `List<String>` | no | Email addresses or names of attendees (internal). |
| | `agendaTopics` | `List<String>` | no | Topics the user wants covered in the brief (1–N). |
| | `requestedBy` | `String` | no | Identity of the requester. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| | `crmSnapshot` | `CrmSnapshot` | no | Raw CRM data. Preserved on entity for audit. |
| `SanitizedCrmData` | `redactedContactName` | `String` | no | e.g., `[REDACTED-NAME]`. |
| | `redactedContactEmail` | `String` | no | e.g., `[REDACTED-EMAIL]`. |
| | `redactedContactPhone` | `String` | no | e.g., `[REDACTED-PHONE]`. |
| | `dealStage` | `String` | no | Passed through unchanged (not PII). |
| | `lastInteractionNote` | `String` | no | Identifiers scrubbed; substance preserved. |
| | `lastContactAt` | `Instant` | no | Passed through unchanged. |
| | `piiCategoriesFound` | `List<String>` | no | e.g., `["person-name", "email", "phone"]`. |
| `TalkingPoint` | `topic` | `String` | no | Maps to one of the submitted agenda topics. |
| | `point` | `String` | no | The substantive content to raise or expect. |
| | `evidenceSource` | `String` | no | Named attachment and date/period, e.g., `"financial-highlights.txt — 10-Q Q1 2026"`. |
| `RiskFlag` | `label` | `String` | no | Short label, e.g., `"Pending litigation"`. |
| | `level` | `RiskLevel` | no | Enum value. |
| | `detail` | `String` | no | One sentence on why this matters for the meeting. |
| `MeetingBrief` | `executiveSummary` | `String` | no | 2–4 sentences on who they are and why the meeting matters now. |
| | `talkingPoints` | `List<TalkingPoint>` | no | 3–5 points ordered by agenda topic. |
| | `riskFlags` | `List<RiskFlag>` | no | 0–3 flags; empty only when attachments are genuinely clean. |
| | `financialHighlights` | `String` | no | 1–2 sentences from the financial attachment. |
| | `recentNewsItems` | `List<String>` | no | 2–4 headline strings from the news attachment. |
| | `preparedAt` | `Instant` | no | When the agent returned. |
| `BriefEval` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `BriefingEvaluator` finished. |
| `Briefing` (entity state) | `briefId` | `String` | no | — |
| | `request` | `Optional<MeetingRequest>` | yes | Populated after `MeetingRequested`. |
| | `sanitized` | `Optional<SanitizedCrmData>` | yes | Populated after `ContactDataSanitized`. |
| | `brief` | `Optional<MeetingBrief>` | yes | Populated after `BriefReady`. |
| | `eval` | `Optional<BriefEval>` | yes | Populated after `BriefEvaluated`. |
| | `status` | `BriefingStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MeetingRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Briefing` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RiskLevel`: `LOW`, `MEDIUM`, `HIGH`.
`BriefingStatus`: `REQUESTED`, `DATA_SANITIZED`, `BRIEFING`, `BRIEF_READY`, `EVALUATED`, `FAILED`.

## Events (`BriefingEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MeetingRequested` | `request` | → REQUESTED |
| `ContactDataSanitized` | `sanitized` | → DATA_SANITIZED |
| `BriefingStarted` | — | → BRIEFING |
| `BriefReady` | `brief` | → BRIEF_READY |
| `BriefEvaluated` | `eval` | → EVALUATED (terminal happy) |
| `BriefingFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Briefing.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BriefingRow` mirrors `Briefing` minus the raw PII fields from `request.crmSnapshot` (`contactName`, `contactEmail`, `contactPhone`). The UI fetches the full brief on demand via `GET /api/briefs/{id}` and reads `request.crmSnapshot` from the JSON when the raw audit trail is needed.

The view declares ONE query: `getAllBriefings: SELECT * AS briefings FROM briefing_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`BriefingTasks.java`)

```java
public final class BriefingTasks {
  public static final Task<MeetingBrief> PREPARE_BRIEF = Task
      .name("Prepare meeting brief")
      .description("Read the sanitized CRM, financial highlights, and news attachments and produce a MeetingBrief for the upcoming meeting")
      .resultConformsTo(MeetingBrief.class);

  private BriefingTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
