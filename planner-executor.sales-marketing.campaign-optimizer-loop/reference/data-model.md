# Data model — campaign-optimizer-loop

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CampaignBrief` | `goal` | `String` | no | User-submitted campaign goal. |
| | `targetAudience` | `String` | no | Audience description from the submission form. |
| | `channels` | `String` | no | Comma-separated channel names. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `CampaignLedger` | `goals` | `List<String>` | no | Goals the planner derived from the brief. |
| | `targetAudience` | `List<String>` | no | Structured audience criteria. |
| | `channelPlan` | `List<String>` | no | Ordered list of execution steps (3–6). |
| | `brandConstraints` | `List<String>` | no | Constraints the planner must honor. |
| | `currentDispatch` | `Optional<DispatchDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `DispatchDecision` | `specialist` | `SpecialistKind` | no | Which specialist runs the step. |
| | `step` | `String` | no | One-sentence step description. |
| | `rationale` | `String` | no | One-sentence justification. |
| `StepResult` | `specialist` | `SpecialistKind` | no | Specialist that ran the step. |
| | `step` | `String` | no | Echo of the step. |
| | `ok` | `boolean` | no | True if the specialist fulfilled the step. |
| | `output` | `String` | no | Raw textual output (pre-guardrail for COPY steps). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `MetricsSnapshot` | `openRate` | `Optional<Double>` | yes | Email open rate (0.0–1.0). Null until PERFORMANCE step. |
| | `clickRate` | `Optional<Double>` | yes | Click-through rate. Null until PERFORMANCE step. |
| | `conversionRate` | `Optional<Double>` | yes | Conversion rate. Null until PERFORMANCE step. |
| | `impressions` | `Optional<Long>` | yes | Total impressions. Null until PERFORMANCE step. |
| `RunEntry` | `attempt` | `int` | no | 1-based attempt count for this `(specialist, step)` pair. |
| | `specialist` | `SpecialistKind` | no | Specialist that ran (or would have run) this step. |
| | `step` | `String` | no | The step text. |
| | `verdict` | `RunVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / KPI_MISS. |
| | `output` | `String` | no | Specialist output text. |
| | `metricsSnapshot` | `MetricsSnapshot` | no | Metrics at time of record (all Optional fields null for non-PERFORMANCE steps). |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `RunLedger` | `entries` | `List<RunEntry>` | no | Append-only. |
| `AudienceSelection` | `segmentName` | `String` | no | Name of the chosen CRM segment. |
| | `estimatedReach` | `long` | no | Estimated audience size from the fixture. |
| | `cohortTags` | `List<String>` | no | Tags describing the segment. |
| `PerformanceAssessment` | `kpisMet` | `boolean` | no | True when all rate metrics meet thresholds. |
| | `current` | `MetricsSnapshot` | no | Observed metrics from the fixture. |
| | `threshold` | `MetricsSnapshot` | no | Declared threshold values. |
| | `recommendations` | `List<String>` | no | Actionable suggestions when `kpisMet = false`. |
| `CampaignReport` | `summary` | `String` | no | 60–120 word campaign summary. |
| | `highlights` | `List<String>` | no | 3–5 cited bullets, each tagged by specialist and step. |
| | `finalMetrics` | `MetricsSnapshot` | no | Metrics at campaign completion. |
| | `producedAt` | `Instant` | no | When the planner produced the report. |
| `ApprovalDecision` | `campaignId` | `String` | no | Campaign this decision applies to. |
| | `approved` | `boolean` | no | True if the marketer approved. |
| | `decidedBy` | `String` | no | Marketer identifier. |
| | `note` | `String` | no | Free-text note from the marketer. |
| | `decidedAt` | `Instant` | no | When the decision was recorded. |
| `Campaign` (entity state) | `campaignId` | `String` | no | Unique id. |
| | `goal` | `String` | no | Original goal from the brief. |
| | `status` | `CampaignStatus` | no | See enum. |
| | `ledger` | `Optional<CampaignLedger>` | yes | Populated after `CampaignPlanned`. |
| | `runLedger` | `Optional<RunLedger>` | yes | Populated after first `StepRecorded` or `StepBlocked`. |
| | `report` | `Optional<CampaignReport>` | yes | Populated after `CampaignCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `CampaignFailed`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `CampaignRejected`. |
| | `createdAt` | `Instant` | no | When `CampaignCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the campaign reached a terminal state. |
| `Approval` (entity state) | `campaignId` | `String` | no | Campaign this entity tracks. |
| | `decision` | `Optional<ApprovalDecision>` | yes | Null until a marketer decides. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(DispatchDecision)`, `Replan(CampaignLedger revised)`, `SubmitForApproval`, `Fail(String reason)`. |

## Enums

- `SpecialistKind` → `COPY`, `AUDIENCE`, `PUBLISH`, `PERFORMANCE`.
- `RunVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `KPI_MISS`.
- `CampaignStatus` → `PLANNING`, `EXECUTING`, `AWAITING_APPROVAL`, `LIVE`, `COMPLETED`, `FAILED`, `REJECTED`.

## Events (`CampaignEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CampaignCreated` | `campaignId, goal, createdAt` | → PLANNING |
| `CampaignPlanned` | `ledger` | → EXECUTING |
| `StepDispatched` | `dispatch` | no status change; sets `ledger.currentDispatch`. |
| `StepBlocked` | `attempt, dispatch, blocker` | no status change; appends a `RunEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `StepRecorded` | `entry: RunEntry` | no status change; appends to `runLedger.entries`. |
| `LedgerRevised` | `ledger: CampaignLedger` | no status change; replaces `ledger`. |
| `ApprovalRequested` | `requestedAt` | → AWAITING_APPROVAL |
| `CampaignApproved` | `decidedBy, note, approvedAt` | → LIVE (combined with `CampaignWentLive`) |
| `CampaignRejected` | `decidedBy, note, rejectedAt` | → REJECTED, `finishedAt = now` |
| `CampaignWentLive` | `wentLiveAt` | → LIVE (emitted with `CampaignApproved`) |
| `PerformanceAlertFired` | `metricsSnapshot, firedAt` | no status change; appends a `RunEntry` with verdict `KPI_MISS` |
| `CampaignCompleted` | `report` | → COMPLETED, `finishedAt = now` |
| `CampaignFailed` | `failureReason` | → FAILED, `finishedAt = now` |

## Events (`ApprovalEntity`)

| Event | Payload |
|---|---|
| `ApprovalGranted` | `campaignId, decidedBy, note, grantedAt` |
| `ApprovalDenied` | `campaignId, decidedBy, note, deniedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `CampaignSubmitted` | `campaignId, goal, targetAudience, channels, requestedBy, submittedAt` |

## View row

`CampaignRow` mirrors `Campaign` minus the heavy run-ledger payload — `runLedger.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `output` is capped at 240 characters. The UI fetches the full campaign by id on click via `GET /api/campaigns/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
