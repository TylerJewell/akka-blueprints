# Data model

Every record, event, enum, and view row the generated system defines.

## `Lead` (LeadEntity state + LeadsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Lead id; equals the workflow id and entity id |
| `companyName` | `Optional<String>` | yes | The submitted company name |
| `contactEmail` | `Optional<String>` | yes | The submitted contact email |
| `status` | `LeadStatus` | no | Lifecycle status |
| `scoredAt` | `Optional<Instant>` | yes | When the score was recorded |
| `score` | `Optional<Integer>` | yes | Numeric ICP score (0–100) |
| `scoreRationale` | `Optional<String>` | yes | Explanation of the score |
| `scoreConfidence` | `Optional<String>` | yes | `high`, `medium`, or `low` |
| `approvedAt` | `Optional<Instant>` | yes | When the reviewer approved |
| `approvedBy` | `Optional<String>` | yes | Reviewer identity |
| `approverNote` | `Optional<String>` | yes | Reviewer note |
| `rejectedAt` | `Optional<Instant>` | yes | When the reviewer rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Rejection reason |
| `qualifiedAt` | `Optional<Instant>` | yes | When qualification completed |
| `qualificationVerdict` | `Optional<String>` | yes | One-sentence qualification verdict |
| `qualificationNextSteps` | `Optional<String>` | yes | Recommended next steps |

Every nullable lifecycle field is `Optional<T>` because `Lead` is the view row type (Lesson 6). `emptyState()` returns `Lead.initial("")` with no `commandContext()` reference (Lesson 3).

## `LeadStatus` enum

`SCORED`, `APPROVED`, `DISQUALIFIED`, `QUALIFIED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `LeadScored` | `recordScore` after ScoringAgent returns | sets `scoredAt`, `score`, `scoreRationale`, `scoreConfidence`, status → `SCORED` |
| `LeadApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `LeadRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `DISQUALIFIED` |
| `LeadQualified` | `recordQualification` after QualificationAgent returns | sets `qualifiedAt`, `qualificationVerdict`, `qualificationNextSteps`, status → `QUALIFIED` |

## Domain records

- `LeadScore(int score, String rationale, String confidence)` — ScoringAgent result.
- `ReviewDecision(String approvedBy, String note)` — approve payload.
- `QualificationSummary(String verdict, String nextSteps)` — QualificationAgent result.

## View row type

`LeadsView` rows are `Lead`. One query: `getAllLeads` → `SELECT * AS leads FROM leads_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (LeadScoringTasks.java)

- `SCORE` — `Task.name(...).resultConformsTo(LeadScore.class)`.
- `QUALIFY` — `Task.name(...).resultConformsTo(QualificationSummary.class)`.
