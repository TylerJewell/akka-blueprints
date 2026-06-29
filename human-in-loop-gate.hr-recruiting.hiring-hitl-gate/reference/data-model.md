# Data model

Every record, event, enum, and view row the generated system defines.

## `Application` (ApplicationEntity state + ApplicationsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Application id; equals the workflow id and entity id |
| `candidateName` | `Optional<String>` | yes | Candidate name as submitted |
| `role` | `Optional<String>` | yes | Target role or job title |
| `status` | `ApplicationStatus` | no | Lifecycle status |
| `submittedAt` | `Optional<Instant>` | yes | When the hiring-request was received |
| `proposedAt` | `Optional<Instant>` | yes | When both agent proposals were recorded |
| `hiringRecommendation` | `Optional<String>` | yes | Recommendation from HiringDecisionProposer |
| `hiringRationale` | `Optional<String>` | yes | Rationale from HiringDecisionProposer |
| `meetingSubject` | `Optional<String>` | yes | Interview invitation subject from MeetingProposer |
| `meetingBody` | `Optional<String>` | yes | Interview invitation body from MeetingProposer |
| `meetingSuggestedSlots` | `Optional<String>` | yes | Suggested time slots from MeetingProposer |
| `approvedAt` | `Optional<Instant>` | yes | When the validator approved |
| `approvedBy` | `Optional<String>` | yes | Validator identity |
| `approverNote` | `Optional<String>` | yes | Validator note |
| `declinedAt` | `Optional<Instant>` | yes | When the validator declined |
| `declinedBy` | `Optional<String>` | yes | Decliner identity |
| `declineReason` | `Optional<String>` | yes | Decline reason |
| `decidedAt` | `Optional<String>` | yes | ISO-8601 outcome-recording time |

Every nullable lifecycle field is `Optional<T>` because `Application` is the view row type (Lesson 6). `emptyState()` returns `Application.initial("")` with no `commandContext()` reference (Lesson 3).

## `ApplicationStatus` enum

`PROPOSED`, `APPROVED`, `DECLINED`, `DECIDED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `ApplicationProposed` | `recordProposals` after both agents return | sets `proposedAt`, `hiringRecommendation`, `hiringRationale`, `meetingSubject`, `meetingBody`, `meetingSuggestedSlots`, status → `PROPOSED` |
| `ApplicationApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `ApplicationDeclined` | `decline` command (human via API) | sets `declinedAt`, `declinedBy`, `declineReason`, status → `DECLINED` |
| `OutcomeRecorded` | `recordOutcome` after DecisionsReachedService step | sets `decidedAt`, status → `DECIDED` |

## Domain records

- `HiringProposal(String recommendation, String rationale)` — HiringDecisionProposer result.
- `MeetingProposal(String subject, String body, String suggestedSlots)` — MeetingProposer result.
- `ValidatorDecision(String approvedBy, String note)` — approve payload.
- `RecordedOutcome(String decidedAt)` — outcome-recording result.

## View row type

`ApplicationsView` rows are `Application`. One query: `getAllApplications` → `SELECT * AS applications FROM applications_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (HiringTasks.java)

- `HIRE_DECISION` — `Task.name(...).resultConformsTo(HiringProposal.class)`.
- `MEETING` — `Task.name(...).resultConformsTo(MeetingProposal.class)`.
