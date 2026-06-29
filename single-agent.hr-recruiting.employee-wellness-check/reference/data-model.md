# Data model — wellness-check-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CheckInQuestion` | `questionId` | `String` | no | Stable id for the question within the campaign. |
| | `text` | `String` | no | The question text shown to the employee. |
| | `kind` | `QuestionKind` | no | `OPEN_ENDED`, `LIKERT_1_5`, or `YES_NO`. |
| `CampaignRequest` | `campaignId` | `String` | no | UUID minted by `CheckInEndpoint`. |
| | `campaignName` | `String` | no | Human-readable label. |
| | `audienceLabel` | `String` | no | Tag identifying the target employee group. |
| | `questions` | `List<CheckInQuestion>` | no | 1–5 questions. |
| | `scheduledAt` | `Instant` | no | When the campaign should activate. |
| | `createdBy` | `String` | no | HR coordinator identifier. |
| `EmployeeResponse` | `checkInId` | `String` | no | UUID minted by `CheckInEndpoint`. |
| | `campaignId` | `String` | no | Parent campaign. |
| | `employeeRef` | `String` | no | Pseudonymised employee reference — not a name or email. |
| | `answers` | `Map<String, String>` | no | `questionId → raw answer text`. Audit-only raw form. |
| | `receivedAt` | `Instant` | no | When the endpoint received the submission. |
| `SanitizedResponse` | `redactedAnswers` | `Map<String, String>` | no | Same keys; values with special-category markers redacted. |
| | `specialCategoriesFound` | `List<String>` | no | e.g. `["mental-health-marker","disability-marker"]`. |
| `CheckInAnalysis` | `moraleLevel` | `MoraleLevel` | no | Enum value. |
| | `interpretation` | `String` | no | 1–3 sentences from the agent. |
| | `crisisFlag` | `boolean` | no | `true` only when `moraleLevel == CRISIS`. |
| | `recommendation` | `String` | no | Actionable verb-phrase from the agent. |
| | `analysedAt` | `Instant` | no | When the agent returned. |
| `SurveillanceResult` | `riskFlagRaised` | `boolean` | no | Whether the campaign's ratio exceeded 0.25. |
| | `rationale` | `String` | no | One sentence naming the measured ratio. |
| | `evaluatedAt` | `Instant` | no | When `MoraleSurveillance` finished. |
| `CheckIn` (entity state) | `checkInId` | `String` | no | — |
| | `campaignId` | `String` | no | — |
| | `response` | `Optional<EmployeeResponse>` | yes | Populated after `ResponseReceived`. |
| | `sanitized` | `Optional<SanitizedResponse>` | yes | Populated after `ResponseSanitized`. |
| | `analysis` | `Optional<CheckInAnalysis>` | yes | Populated after `AnalysisRecorded`. |
| | `surveillance` | `Optional<SurveillanceResult>` | yes | Populated after `SurveillanceEvaluated`. |
| | `status` | `CheckInStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ResponseReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `Campaign` (entity state) | `campaignId` | `String` | no | — |
| | `request` | `Optional<CampaignRequest>` | yes | Populated after `CampaignScheduled`. |
| | `status` | `CampaignStatus` | no | See enum. |
| | `responsesReceived` | `int` | no | Total check-ins received. |
| | `highCount` | `int` | no | Count of HIGH morale analyses. |
| | `moderateCount` | `int` | no | Count of MODERATE morale analyses. |
| | `lowCount` | `int` | no | Count of LOW morale analyses. |
| | `crisisCount` | `int` | no | Count of CRISIS / escalated check-ins. |
| | `createdAt` | `Instant` | no | When `CampaignScheduled` emitted. |
| | `completedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `CheckIn` and `Campaign` is `Optional<T>`. The view table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QuestionKind`: `OPEN_ENDED`, `LIKERT_1_5`, `YES_NO`.
`MoraleLevel`: `HIGH`, `MODERATE`, `LOW`, `CRISIS`.
`CheckInStatus`: `RECEIVED`, `SANITIZED`, `ANALYSING`, `ANALYSIS_RECORDED`, `ESCALATED`, `EVALUATED`, `FAILED`.
`CampaignStatus`: `SCHEDULED`, `ACTIVE`, `COMPLETED`, `FAILED`.

## Events (`CheckInEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ResponseReceived` | `response` | → RECEIVED |
| `ResponseSanitized` | `sanitized` | → SANITIZED |
| `AnalysisStarted` | — | → ANALYSING |
| `AnalysisRecorded` | `analysis` | → ANALYSIS_RECORDED |
| `CrisisEscalated` | `reason: String` | → ESCALATED (terminal crisis) |
| `SurveillanceEvaluated` | `surveillance` | → EVALUATED (terminal happy) |
| `CheckInFailed` | `reason: String` | → FAILED (terminal error) |

## Events (`CampaignEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CampaignScheduled` | `request` | → SCHEDULED |
| `CampaignActivated` | — | → ACTIVE |
| `CampaignCompleted` | — | → COMPLETED |
| `CampaignFailed` | `reason: String` | → FAILED |

`emptyState()` on both entities returns the respective `.initial()` factory with all `Optional` fields as `Optional.empty()`. Neither references `commandContext()` in `emptyState()` (Lesson 3).

## View rows

`CheckInRow` mirrors `CheckIn` minus `response.answers` raw values — the audit log keeps those. The UI fetches raw answers on demand via `GET /api/check-ins/{id}` and reads `response.answers` from the JSON.

`CampaignRow` mirrors `Campaign` in full (no sensitive content to exclude). Its dual subscription (campaign events + check-in `AnalysisRecorded` events) lets the morale count columns stay current without a join query.

Each view declares ONE query returning all rows. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`CheckInTasks.java`)

```java
public final class CheckInTasks {
  public static final Task<CheckInAnalysis> ANALYSE_CHECK_IN = Task
      .name("Analyse check-in response")
      .description("Read the attached sanitized employee response and produce a CheckInAnalysis "
          + "with morale level, interpretation, crisis flag, and recommendation")
      .resultConformsTo(CheckInAnalysis.class);

  private CheckInTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
