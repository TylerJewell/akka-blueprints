# Data model — `recruitment-team`

Authoritative record and event definitions. `/akka:implement` writes these as specified.

## `Candidate` (CandidateEntity state + CandidatesView row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Candidate / workflow id |
| `roleId` | `String` | no | Role applied for |
| `sanitizedResume` | `Optional<String>` | yes | Resume after protected-attribute redaction |
| `status` | `CandidateStatus` | no | Lifecycle state |
| `matchScore` | `Optional<Integer>` | yes | 0–100 fit score from MatchAgent |
| `matchSummary` | `Optional<String>` | yes | MatchAgent fit summary |
| `screenRecommendation` | `Optional<String>` | yes | `ADVANCE` / `HOLD` / `REJECT` |
| `screenReasons` | `Optional<String>` | yes | ScreenAgent reasons |
| `evalScore` | `Optional<Integer>` | yes | E1 eval score on the screen decision |
| `supervisorSummary` | `Optional<String>` | yes | SupervisorAgent reviewer brief |
| `decidedBy` | `Optional<String>` | yes | Human who approved/rejected |
| `decisionReason` | `Optional<String>` | yes | Reject reason |
| `sanitizedAt` | `Optional<String>` | yes | ISO-8601 timestamp |
| `screenedAt` | `Optional<String>` | yes | ISO-8601 timestamp |
| `decidedAt` | `Optional<String>` | yes | ISO-8601 timestamp |
| `escalatedAt` | `Optional<String>` | yes | ISO-8601 timestamp |
| `driftFlagged` | `Optional<Boolean>` | yes | Set by FairnessDriftMonitor |

`emptyState()` returns `Candidate.initial("", "")` with no `commandContext()` reference (Lesson 3). Every nullable field is `Optional<T>` so the view materializer accepts the row (Lesson 6).

## `CandidateStatus` enum

`RECEIVED, SANITIZED, MATCHED, SCREENED, AWAITING_DECISION, APPROVED, REJECTED, ESCALATED`

## Typed agent results

| Record | Fields |
|---|---|
| `CandidateMatch` | `int score, String summary, List<String> matchedSkills` |
| `ScreenDecision` | `String recommendation, String reasons` |
| `SupervisorSummary` | `String recommendation, String summary` |

## Events (CandidateEntity)

| Event | Trigger |
|---|---|
| `ResumeSanitized` | `sanitizeStep` finishes redaction |
| `CandidateMatched` | MatchAgent returns |
| `CandidateScreened` | ScreenAgent returns |
| `DecisionEvaluated` | E1 eval scores the screen decision |
| `DecisionRequested` | SupervisorAgent done; candidate parked for review |
| `CandidateApproved` | Human approve command |
| `CandidateRejected` | Human reject command |
| `CandidateEscalated` | StuckDecisionMonitor escalation |
| `DriftFlagged` | FairnessDriftMonitor crosses threshold |

## Events (InboundApplicationQueue)

| Event | Trigger |
|---|---|
| `ApplicationQueued` | `enqueueApplication(roleId, resume)` from endpoint or simulator |

## View

`CandidatesView` — row type `Candidate`, one query `getAllCandidates` (`SELECT * AS candidates FROM candidates_view`). No `WHERE status` filter; callers filter client-side (Lesson 2).
