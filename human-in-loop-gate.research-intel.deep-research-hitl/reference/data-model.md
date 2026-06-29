# Data model

Every record, event, enum, and view row the generated system defines.

## `Report` (ResearchEntity state + ReportsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Report id; equals the workflow id and entity id |
| `query` | `Optional<String>` | yes | The submitted research query |
| `status` | `ReportStatus` | no | Lifecycle status |
| `planCreatedAt` | `Optional<Instant>` | yes | When the research plan was recorded |
| `subTopics` | `Optional<List<String>>` | yes | Sub-topics produced by PlannerAgent |
| `findingsSummaries` | `Optional<List<String>>` | yes | Per-sub-topic summary strings from ResearchAgent |
| `synthesisStartedAt` | `Optional<Instant>` | yes | When SynthesisAgent was invoked |
| `reportTitle` | `Optional<String>` | yes | Report headline from SynthesisAgent |
| `reportBody` | `Optional<String>` | yes | Full synthesised report body |
| `sourcesUsed` | `Optional<List<String>>` | yes | Source URLs cited in the report body |
| `awaitingReviewAt` | `Optional<Instant>` | yes | When the report entered AWAITING_REVIEW |
| `approvedAt` | `Optional<Instant>` | yes | When the reviewer approved |
| `approvedBy` | `Optional<String>` | yes | Reviewer identity (approve) |
| `approverNotes` | `Optional<String>` | yes | Reviewer notes on approval |
| `rejectedAt` | `Optional<Instant>` | yes | When the reviewer rejected |
| `rejectedBy` | `Optional<String>` | yes | Reviewer identity (reject) |
| `revisionNotes` | `Optional<String>` | yes | Reviewer feedback driving re-synthesis |
| `deliveredAt` | `Optional<String>` | yes | ISO-8601 delivery timestamp |

Every nullable lifecycle field is `Optional<T>` because `Report` is the view row type (Lesson 6). `emptyState()` returns `Report.initial("")` with no `commandContext()` reference (Lesson 3).

## `ReportStatus` enum

`PLANNING`, `INVESTIGATING`, `SYNTHESISING`, `AWAITING_REVIEW`, `APPROVED`, `NEEDS_REVISION`, `DELIVERED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `QueryPlanned` | `recordPlan` after PlannerAgent returns | sets `planCreatedAt`, `subTopics`, status → `PLANNING` |
| `InvestigationStarted` | `recordFindings` first call | status → `INVESTIGATING` |
| `FindingsRecorded` | `recordFindings` each subsequent call | appends to `findingsSummaries` |
| `SynthesisStarted` | `recordDraft` command issued | sets `synthesisStartedAt`, status → `SYNTHESISING` |
| `ReportAwaitingReview` | `recordDraft` after SynthesisAgent returns | sets `reportTitle`, `reportBody`, `sourcesUsed`, `awaitingReviewAt`, status → `AWAITING_REVIEW` |
| `ReviewApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverNotes`, status → `APPROVED` |
| `ReviewRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `revisionNotes`, status → `NEEDS_REVISION` |
| `ReportDelivered` | `recordDelivery` after deliver step | sets `deliveredAt`, status → `DELIVERED` |

## Domain records

- `ResearchPlan(String queryId, List<String> subTopics)` — PlannerAgent result.
- `SubTopicFindings(String subTopic, String summary, List<String> sources)` — ResearchAgent result.
- `ReportDraft(String title, String body, List<String> sourcesUsed)` — SynthesisAgent result.
- `ReviewDecision(String reviewedBy, String notes)` — approve / reject payload.
- `DeliveredReport(String reportId, String deliveredAt)` — delivery record.

## View row type

`ReportsView` rows are `Report`. One query: `getAllReports` → `SELECT * AS reports FROM reports_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ResearchTasks.java)

- `PLAN` — `Task.name(...).resultConformsTo(ResearchPlan.class)`.
- `INVESTIGATE` — `Task.name(...).resultConformsTo(SubTopicFindings.class)`.
- `SYNTHESISE` — `Task.name(...).resultConformsTo(ReportDraft.class)`.
