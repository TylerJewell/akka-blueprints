# Data model

Every record, event, enum, and view row the generated system defines.

## `Job` (JobEntity state + JobsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Job id; equals the workflow id and entity id |
| `requestPayload` | `Optional<String>` | yes | The submitted job request string |
| `status` | `JobStatus` | no | Lifecycle status |
| `analyzedAt` | `Optional<Instant>` | yes | When the analysis was recorded |
| `findings` | `Optional<String>` | yes | Analysis findings text |
| `riskLevel` | `Optional<String>` | yes | Risk classification: LOW, MEDIUM, or HIGH |
| `approvedAt` | `Optional<Instant>` | yes | When the job was approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverNote` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When the job was rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Rejection reason |
| `completedAt` | `Optional<String>` | yes | ISO-8601 completion time |
| `output` | `Optional<String>` | yes | Completion output description |

Every nullable lifecycle field is `Optional<T>` because `Job` is the view row type (Lesson 6). `emptyState()` returns `Job.initial("")` with no `commandContext()` reference (Lesson 3).

## `JobStatus` enum

`ANALYZED`, `APPROVED`, `REJECTED`, `COMPLETED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `JobAnalyzed` | `recordAnalysis` after AnalysisAgent returns | sets `analyzedAt`, `findings`, `riskLevel`, status → `ANALYZED` |
| `JobApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `JobRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `JobCompleted` | `recordCompletion` after CompletionAgent returns | sets `completedAt`, `output`, status → `COMPLETED` |

## Domain records

- `AnalysisSummary(String findings, String riskLevel)` — AnalysisAgent result.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `JobResult(String output, String completedAt)` — CompletionAgent result.

## View row type

`JobsView` rows are `Job`. One query: `getAllJobs` → `SELECT * AS jobs FROM jobs_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (JobTasks.java)

- `ANALYZE` — `Task.name(...).resultConformsTo(AnalysisSummary.class)`.
- `COMPLETE` — `Task.name(...).resultConformsTo(JobResult.class)`.
