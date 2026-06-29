# Data model — Fair CV Matcher

Every record the generated system defines. Nullable lifecycle fields are `Optional<T>` (Lesson 6); Akka serializes `Optional` transparently as the raw value or `null`.

## Records

### CvProfile (extraction output)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| yearsExperience | int | no | Inferred total years of experience |
| skills | List<String> | no | Concrete job-relevant skills |
| education | String | no | Highest completed qualification |
| currentTitle | String | no | Most recent role title |

### RedactedSignal (one stripped signal)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| field | String | no | Profile field the value came from |
| category | String | no | Protected category (name, address, age, gender, photo) |

### MatchResult (one scored posting)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| jobId | String | no | Posting id |
| jobTitle | String | no | Posting title |
| score | int | no | Fit score 0–100 |
| rationale | String | no | One-sentence justification |

### Candidate (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | String | no | Candidate id |
| status | CandidateStatus | no | Lifecycle state |
| submittedAt | Instant | no | Submission time |
| slice | Optional<String> | yes | Declared demographic slice for fairness grouping |
| profile | Optional<CvProfile> | yes | Set at extraction |
| redactions | List<RedactedSignal> | no (may be empty) | Set at sanitization |
| matches | List<MatchResult> | no (may be empty) | Set at matching |
| extractedAt | Optional<Instant> | yes | Set at extraction |
| sanitizedAt | Optional<Instant> | yes | Set at sanitization |
| matchedAt | Optional<Instant> | yes | Set at matching |
| reviewedAt | Optional<Instant> | yes | Set at review |
| reviewedBy | Optional<String> | yes | Reviewer id |
| reviewNote | Optional<String> | yes | Reviewer note |

`emptyState()` returns `Candidate.initial("")` with no `commandContext()` reference (Lesson 3).

## Status enum

`CandidateStatus { SUBMITTED, EXTRACTED, SANITIZED, MATCHED, REVIEWED }`

## Events (CandidateEntity)

| Event | Trigger |
|---|---|
| CvSubmitted | A submission is accepted; candidate created in SUBMITTED |
| ProfileExtracted | ExtractionAgent returns a CvProfile |
| ProfileSanitized | Sanitize step strips special-category signals |
| MatchesScored | MatchingAgent returns the ranked matches |
| ReviewRecorded | A reviewer posts oversight |

`CvIntakeQueue` emits `CvSubmittedToQueue` on `enqueue`.

## View row type

`MatchesView` rows are `Candidate` records. One query: `getAllCandidates` → `SELECT * AS candidates FROM matches_view`. No `WHERE status` filter — Akka cannot auto-index the enum column (Lesson 2); status filtering is client-side in callers.
