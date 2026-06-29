# Data model — sdr-lead-nurture

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `LeadContact` | `leadId` | `String` | no | UUID minted by `LeadEndpoint`. |
| | `firstName` | `String` | no | Raw first name supplied by the ingest form. |
| | `lastName` | `String` | no | Raw last name supplied by the ingest form. |
| | `company` | `String` | no | Company or organization name. |
| | `jobTitle` | `String` | no | Job title or role. |
| | `email` | `String` | no | Raw email address. Audit-only after sanitization. |
| | `phone` | `String` | no | Raw phone number; may be empty string. |
| | `channel` | `InboundChannel` | no | How the lead arrived. |
| | `initialMessage` | `String` | no | The lead's first message text. |
| | `receivedAt` | `Instant` | no | When `LeadEndpoint` ingested the record. |
| `SanitizedContact` | `redactedRecord` | `String` | no | JSON string of the contact with PII replaced by tokens. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","person-name","address"]`. |
| `ConversationTurn` | `turnId` | `String` | no | UUID per turn. |
| | `role` | `TurnRole` | no | `LEAD` or `AGENT`. |
| | `message` | `String` | no | The message text. |
| | `sentAt` | `Instant` | no | When the turn was recorded. |
| `AgentTurn` | `message` | `String` | no | The agent's outbound reply text. |
| | `toolCall` | `Optional<ToolCall>` | yes | Present only on a close-action turn. |
| | `decidedAt` | `Instant` | no | When `SdrAgent` returned the turn. |
| `ToolCall` | `tool` | `ToolName` | no | Which tool to call. |
| | `params` | `Map<String, Object>` | no | Tool-specific parameter map. |
| `MeetingBooking` | `slot` | `String` | no | ISO-8601 datetime of the meeting. |
| | `durationMinutes` | `int` | no | 15, 30, or 45. |
| | `accountExecutiveId` | `String` | no | AE assigned to the meeting. |
| | `zoomLink` | `String` | no | Meeting URL (stub-generated in the blueprint). |
| `CrmUpdate` | `newStatus` | `LeadStatus` | no | Target status for the CRM record. |
| | `note` | `String` | no | Optional note written to the CRM. |
| `QualityScore` | `score` | `int` | no | 1–5 from `QualityScorer`. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `Lead` (entity state) | `leadId` | `String` | no | — |
| | `contact` | `Optional<LeadContact>` | yes | Populated after `LeadReceived`. |
| | `sanitized` | `Optional<SanitizedContact>` | yes | Populated after `LeadSanitized`. |
| | `conversation` | `List<ConversationTurn>` | no | Grows with each `TurnRecorded` event. |
| | `booking` | `Optional<MeetingBooking>` | yes | Populated after `MeetingBooked`. |
| | `quality` | `Optional<QualityScore>` | yes | Populated after `QualityScoredEvent`. |
| | `status` | `LeadStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `LeadReceived` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Lead` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`InboundChannel`: `WEBSITE_CHAT`, `EMAIL`, `EVENT`, `REFERRAL`.

`TurnRole`: `LEAD`, `AGENT`.

`ToolName`: `BOOK_MEETING`, `UPDATE_CRM_STATUS`, `DISMISS_LEAD`, `HANDOFF_TO_AE`.

`LeadStatus`: `RECEIVED`, `SANITIZED`, `ENGAGING`, `MEETING_BOOKED`, `DISMISSED`, `HANDED_OFF`, `FAILED`.

## Events (`LeadEntity`)

| Event | Payload | Transition |
|---|---|---|
| `LeadReceived` | `contact: LeadContact` | → RECEIVED |
| `LeadSanitized` | `sanitized: SanitizedContact` | → SANITIZED |
| `EngagementStarted` | — | → ENGAGING |
| `TurnRecorded` | `turn: ConversationTurn` | stays ENGAGING |
| `MeetingBooked` | `booking: MeetingBooking` | → MEETING_BOOKED |
| `LeadDismissed` | `reason: String` | → DISMISSED |
| `HandedOffToAE` | `note: String` | → HANDED_OFF |
| `QualityScoredEvent` | `quality: QualityScore` | terminal state unchanged |
| `LeadFailed` | `reason: String` | → FAILED |

`emptyState()` returns `Lead.initial("")` with `conversation = List.of()`, all `Optional` fields as `Optional.empty()`, and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`LeadRow` mirrors `Lead` minus `contact.email` and `contact.phone` (the audit log retains raw PII; the view holds the sanitized form only). The conversation list is included in the row so the UI can render the full thread without a separate fetch.

The view declares ONE query: `getAllLeads: SELECT * AS leads FROM lead_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`LeadTasks.java`)

```java
public final class LeadTasks {
  public static final Task<AgentTurn> SDR_ENGAGE = Task
      .name("Engage lead")
      .description("Nurture the inbound lead through qualification; reply to their message or call a close-action tool")
      .resultConformsTo(AgentTurn.class);

  private LeadTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
