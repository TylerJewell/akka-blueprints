# Data model — post-incident-reviewer

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncidentRecord` | `incidentId` | `String` | no | Canonical incident identifier (e.g., `INC-2026-0741`). |
| | `title` | `String` | no | Short incident title. |
| | `severity` | `String` | no | One of `P1`, `P2`, `P3`, `P4`. |
| | `reportedBy` | `String` | no | Email or name of the person who reported the incident. |
| | `detectedAt` | `Instant` | no | When monitoring or a human first detected the incident. |
| | `resolvedAt` | `Instant` | no | When the incident was declared resolved. |
| `TimelineEvent` | `eventId` | `String` | no | Short stable id (e.g., `evt-001`). MUST be unique within an `EvidenceLog`. |
| | `occurredAt` | `Instant` | no | When the event occurred. |
| | `actor` | `String` | no | Person, system, or tool that produced the event. |
| | `description` | `String` | no | Human-readable description of what happened. |
| `EvidenceLog` | `incident` | `IncidentRecord` | no | The incident header record. |
| | `timeline` | `List<TimelineEvent>` | no | Possibly empty for unknown incident IDs (J6). |
| | `gatheredAt` | `Instant` | no | When the GATHER task returned. |
| `ImpactClassification` | `severity` | `String` | no | MUST equal `EvidenceLog.incident.severity` (G1 check 4). |
| | `affectedSystems` | `String` | no | Comma-separated names of affected services or tiers. |
| | `usersAffected` | `int` | no | Estimated number of users affected. |
| | `outageWindow` | `Duration` | no | `resolvedAt - detectedAt`. |
| `RootCause` | `rootCauseId` | `String` | no | Short stable id (e.g., `rc-b9f2`). |
| | `summary` | `String` | no | 1–2 sentence root-cause description. |
| | `contributingFactors` | `List<String>` | no | Possibly empty; each factor is a short noun phrase. |
| `ImpactAssessment` | `classification` | `ImpactClassification` | no | — |
| | `rootCause` | `RootCause` | no | — |
| | `assessedAt` | `Instant` | no | When the ASSESS task returned. |
| `ActionItem` | `actionId` | `String` | no | Short stable id (e.g., `ai-01`). |
| | `description` | `String` | no | What must be done. |
| | `owner` | `String` | no | Non-null (G1 check 2). Email or name of the responsible party. |
| | `dueDate` | `LocalDate` | no | Non-null (G1 check 2). |
| | `priority` | `String` | no | One of `HIGH`, `MEDIUM`, `LOW`. |
| `PostIncidentReview` | `pirId` | `String` | no | Matches the `PIRRecord.pirId`. |
| | `executiveSummary` | `String` | no | Non-empty, ≥ 50 chars (G1 check 1). Or the exact refusal string for empty-evidence path. |
| | `impactClassification` | `ImpactClassification` | no | Severity MUST match `EvidenceLog.incident.severity` (G1 check 4). |
| | `timeline` | `List<TimelineEvent>` | no | A subset of `EvidenceLog.timeline`; every `eventId` MUST be present there (G1 check 3). |
| | `rootCause` | `RootCause` | no | — |
| | `actionItems` | `List<ActionItem>` | no | `size() >= 1` except on the refusal path (G1 check 2). |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned and the guardrail accepted. |
| `SignoffDecision` | `approved` | `boolean` | no | `true` → COMPLETE; `false` → REJECTED. |
| | `comments` | `Optional<String>` | yes | Freeform comment from the incident owner. |
| | `decidedBy` | `String` | no | Identity of the person who submitted the sign-off. |
| | `decidedAt` | `Instant` | no | When the sign-off was recorded. |
| `GuardrailRejection` | `check` | `String` | no | Which of the four G1 checks failed (e.g., `"action-item-owner"`). |
| | `reason` | `String` | no | Structured reason from `PIRGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `PIRRecord` (entity state) | `pirId` | `String` | no | — |
| | `incidentId` | `Optional<String>` | yes | Populated after `PIRCreated`. |
| | `evidenceLog` | `Optional<EvidenceLog>` | yes | Populated after `EvidenceGathered`. |
| | `impactAssessment` | `Optional<ImpactAssessment>` | yes | Populated after `ImpactAssessed`. |
| | `review` | `Optional<PostIncidentReview>` | yes | Populated after `ReviewDrafted`. |
| | `signoffDecision` | `Optional<SignoffDecision>` | yes | Populated after `SignoffRecorded`. |
| | `status` | `PIRStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PIRCreated` emitted. |
| | `completedAt` | `Optional<Instant>` | yes | Terminal-state timestamp (COMPLETE or REJECTED). |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `PIRRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PIRStatus`: `CREATED`, `GATHERING`, `GATHERED`, `ASSESSING`, `ASSESSED`, `DRAFTING`, `DRAFTED`, `AWAITING_SIGNOFF`, `COMPLETE`, `REJECTED`, `FAILED`.

## Events (`PIREntity`)

| Event | Payload | Transition |
|---|---|---|
| `PIRCreated` | `incidentId: String` | → CREATED |
| `GatherStarted` | — | → GATHERING |
| `EvidenceGathered` | `evidenceLog: EvidenceLog` | → GATHERED |
| `AssessStarted` | — | → ASSESSING |
| `ImpactAssessed` | `impactAssessment: ImpactAssessment` | → ASSESSED |
| `DraftStarted` | — | → DRAFTING |
| `ReviewDrafted` | `review: PostIncidentReview` | → DRAFTED |
| `GuardrailRejected` | `check, reason, rejectedAt` | no status change (audit-only) |
| `SignoffRequested` | `assigneeEmail: String` | → AWAITING_SIGNOFF |
| `SignoffRecorded` | `decision: SignoffDecision` | no status change (decision stored) |
| `ReviewComplete` | — | → COMPLETE (terminal happy) |
| `ReviewRejected` | `rejectionComment: Optional<String>` | → REJECTED (terminal) |
| `PIRFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `PIRRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PIRRow` mirrors `PIRRecord` exactly. The UI fetches the full row via `GET /api/pir/{id}` and streams updates via `GET /api/pir/sse`.

The view declares ONE query: `getAllPIRs: SELECT * AS reviews FROM pir_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`PIRTasks.java`)

```java
public final class PIRTasks {
  public static final Task<EvidenceLog> GATHER_EVIDENCE = Task
      .name("Gather evidence")
      .description("Fetch the incident record and all timeline events for this incident")
      .resultConformsTo(EvidenceLog.class);

  public static final Task<ImpactAssessment> ASSESS_IMPACT = Task
      .name("Assess impact")
      .description("Classify impact severity and identify the root cause from the evidence log")
      .resultConformsTo(ImpactAssessment.class);

  public static final Task<PostIncidentReview> DRAFT_REVIEW = Task
      .name("Draft review")
      .description("Compose a PostIncidentReview with executive summary, timeline, root cause, and action items")
      .resultConformsTo(PostIncidentReview.class);

  private PIRTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Guardrail checks

`PIRGuardrail` applies four checks on every DRAFT task response before `ReviewDrafted` is committed:

| Check ID | Field | Rule |
|---|---|---|
| `executive-summary` | `executiveSummary` | Non-null and `length() >= 50` (or equals the exact refusal string). |
| `action-item-owner` | `actionItems` | `size() >= 1`; every item has non-null `owner` and non-null `dueDate`. |
| `timeline-reference` | `timeline[].eventId` | Every `eventId` in the review's `timeline` must appear in `EvidenceLog.timeline`. |
| `severity-consistency` | `impactClassification.severity` | Must equal `EvidenceLog.incident.severity`. |

GATHER and ASSESS task responses pass through `PIRGuardrail` immediately without any check — the guardrail reads the current task name from the response context.
