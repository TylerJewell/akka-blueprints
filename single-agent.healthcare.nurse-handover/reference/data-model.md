# Data model — nurse-handover

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChecklistItem` | `itemId` | `String` | no | Stable id supplied by the submitting clinician. |
| | `description` | `String` | no | What the incoming team must check. |
| | `urgency` | `String` | no | `ROUTINE`, `URGENT`, or `CRITICAL`. |
| `HandoverRequest` | `handoverId` | `String` | no | UUID minted by `HandoverEndpoint`. |
| | `ward` | `String` | no | Ward name (e.g., "General Medicine"). |
| | `shiftDate` | `String` | no | ISO-8601 date string (`YYYY-MM-DD`). |
| | `submittedBy` | `String` | no | Submitting clinician identifier. |
| | `rawReport` | `String` | no | Pre-sanitization shift report body. Audit-only. |
| | `checklist` | `List<ChecklistItem>` | no | Submitted checklist (1–N items). |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedReport` | `redactedReport` | `String` | no | PHI redacted; this is what the agent sees. |
| | `phiCategoriesFound` | `List<String>` | no | e.g. `["mrn","dob","phone","insurance-id"]`. |
| `PatientStatus` | `patientRef` | `String` | no | De-identified label from the report (e.g. "Patient-A"). |
| | `currentCondition` | `String` | no | 1–2 sentence current status. |
| | `outstandingTasks` | `String` | no | Comma-separated tasks specific to this patient. |
| | `riskLevel` | `RiskLevel` | no | Enum value. |
| `HandoverSummary` | `patients` | `List<PatientStatus>` | no | One entry per patient reference in the report. |
| | `outstandingTasks` | `List<String>` | no | Ward-level tasks, urgency-ordered. |
| | `riskFlags` | `List<String>` | no | Items needing immediate attention. |
| | `narrativeSummary` | `String` | no | 2–4 sentences for the incoming clinician. |
| | `summarizedAt` | `Instant` | no | When the agent returned. |
| `SignoffRecord` | `signedOffBy` | `String` | no | Attesting clinician identifier. |
| | `signedOffAt` | `Instant` | no | When the signoff PATCH was processed. |
| `Handover` (entity state) | `handoverId` | `String` | no | — |
| | `request` | `Optional<HandoverRequest>` | yes | Populated after `ReportSubmitted`. |
| | `sanitized` | `Optional<SanitizedReport>` | yes | Populated after `ReportSanitized`. |
| | `summary` | `Optional<HandoverSummary>` | yes | Populated after `SummaryRecorded`. |
| | `signoff` | `Optional<SignoffRecord>` | yes | Populated after `HandoverSignedOff`. |
| | `status` | `HandoverStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ReportSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Handover` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RiskLevel`: `LOW`, `MODERATE`, `HIGH`, `CRITICAL`.
`HandoverStatus`: `SUBMITTED`, `SANITIZED`, `SUMMARIZING`, `SUMMARY_READY`, `AWAITING_SIGNOFF`, `SIGNED_OFF`, `FAILED`.

## Events (`HandoverEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ReportSubmitted` | `request` | → SUBMITTED |
| `ReportSanitized` | `sanitized` | → SANITIZED |
| `SummarizationStarted` | — | → SUMMARIZING |
| `SummaryRecorded` | `summary` | → SUMMARY_READY |
| `SignoffRequested` | `signedOffBy: String` | → AWAITING_SIGNOFF |
| `HandoverSignedOff` | `signoff` | → SIGNED_OFF (terminal happy) |
| `HandoverFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Handover.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`HandoverRow` mirrors `Handover` minus `request.rawReport` (the audit log keeps that). The UI fetches the raw report on demand via `GET /api/handovers/{id}` and reads `request.rawReport` from the JSON.

The view declares ONE query: `getAllHandovers: SELECT * AS handovers FROM handover_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`HandoverTasks.java`)

```java
public final class HandoverTasks {
  public static final Task<HandoverSummary> SUMMARIZE_HANDOVER = Task
      .name("Summarize handover")
      .description("Read the attached sanitized shift report and produce a HandoverSummary per ward checklist item")
      .resultConformsTo(HandoverSummary.class);

  private HandoverTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
