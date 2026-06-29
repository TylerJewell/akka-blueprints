# Data model

Every nullable lifecycle field is `Optional<T>` on the view row record (Lesson 6).

## `Candidate` (entity state + `CandidatesView` row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Candidate evaluation id (UUID) |
| `requisitionId` | `Optional<String>` | yes | Originating requisition |
| `roleTitle` | `Optional<String>` | yes | Role being filled |
| `status` | `CandidateStatus` | no | Lifecycle state |
| `sourcedAt` | `Optional<Instant>` | yes | When the profile was sourced |
| `sourceHandle` | `Optional<String>` | yes | Anonymized source handle |
| `sanitizedAt` | `Optional<Instant>` | yes | When redaction ran |
| `redactedCategories` | `Optional<String>` | yes | Categories removed by the sanitizer |
| `screenedAt` | `Optional<Instant>` | yes | When screening ran |
| `screenSummary` | `Optional<String>` | yes | Screening evidence summary |
| `screenPass` | `Optional<Boolean>` | yes | Whether screening passed |
| `matchedAt` | `Optional<Instant>` | yes | When scoring ran |
| `matchScore` | `Optional<Double>` | yes | 0–100 fit score |
| `matchRationale` | `Optional<String>` | yes | Rationale for the score |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectReason` | `Optional<String>` | yes | Why rejected |
| `blockedAt` | `Optional<Instant>` | yes | When sourcing was blocked |
| `blockedDomain` | `Optional<String>` | yes | The non-allowlisted domain |
| `haltedAt` | `Optional<Instant>` | yes | When halted |

`emptyState()` returns `Candidate.initial("")` — placeholder id, `status = SOURCED`
will be set by the first event; no `commandContext()` reference (Lesson 3).

## `CandidateStatus` enum

`SOURCED, SANITIZED, SCREENED, MATCHED, REJECTED, BLOCKED, HALTED`

## Events (`CandidateEntity`)

| Event | Trigger |
|---|---|
| `CandidateSourced` | `SourcingAgent` returned a `SourcedProfile` |
| `ProfileSanitized` | `ProfileSanitizer.redact` ran in `sanitizeStep` |
| `CandidateScreened` | `ScreeningAgent` returned a `ScreeningResult` |
| `CandidateMatched` | `MatchingAgent` returned a `MatchResult` (pass) |
| `CandidateRejected` | Screening failed or a step exhausted retries |
| `SourcingBlocked` | The before-tool-call guardrail blocked the domain |
| `EvaluationHalted` | A halt was active at the workflow's first check |

## Other records

| Record | Fields |
|---|---|
| `SourcedProfile` | `String handle`, `String rawText` |
| `ScreeningResult` | `boolean pass`, `String summary` |
| `MatchResult` | `double score`, `String rationale` |
| `Requisition` | `String roleTitle`, `String requirements`, `int candidateCount`, `boolean forceBlockedDomain` |
| `MonitoringSnapshot` | `int evaluated`, `int matched`, `int rejected`, `int blocked`, `double avgScore` |
| `SystemControl` (KVE state) | `boolean halted` |

## `RequisitionQueue` events

| Event | Trigger |
|---|---|
| `RequisitionQueued` | `RecruitmentEndpoint` or `RequisitionSimulator` enqueued a `Requisition` |

## `RecruitmentTasks` (companion for the AutonomousAgents — Lesson 7)

| Task constant | `resultConformsTo` |
|---|---|
| `SOURCE` | `SourcedProfile` |
| `SCREEN` | `ScreeningResult` |
| `MATCH` | `MatchResult` |
