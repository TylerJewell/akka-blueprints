# Data model

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| ResearchQuestion | inquiryId | String | no | Id assigned at submission. |
| | question | String | no | Free-text research question. |
| | submittedBy | String | no | UI identifier of the submitter. |
| Source | sourceId | String | no | Entity-assigned id. |
| | citation | String | no | Bibliographic citation. |
| | url | String | no | Source URL. |
| | addedAt | Instant | no | When committed. |
| Claim | claimId | String | no | Deterministic id `inquiryId + ":c:" + index`. |
| | text | String | no | One-sentence factual statement. |
| | supports | List<String> | no | Sub-question texts this claim supports (may be empty). |
| | derivedFrom | String | no | `sourceId` of the supporting source. |
| | confidence | double | no | `[0.0, 1.0]`. |
| Hypothesis | hypothesisId | String | no | Deterministic id `inquiryId + ":h:" + index`. |
| | statement | String | no | One-sentence answer to the question. |
| | backedByClaims | List<String> | no | Non-empty list of `claimId` values. |
| | proposedAt | Instant | no | When committed. |
| Challenge | challengeId | String | no | Deterministic id. |
| | targetId | String | no | Id of the claim or hypothesis under challenge. |
| | targetKind | String | no | `"claim"` or `"hypothesis"`. |
| | rationale | String | no | One-sentence statement of the weakness. |
| | raisedAt | Instant | no | When raised. |
| ProposedWrite | specialist | String | no | Specialist class name. |
| | writeKind | String | no | `"source-batch"` / `"claim-batch"` / `"hypothesis"` / `"challenge"`. |
| | payload | Object | no | Type-specific payload validated by `SchemaValidator`. |
| ContributionScore | score | double | no | Utility score in `[0.0, 1.0]`. |
| | rationale | String | no | One-sentence justification. |
| | scoredAt | Instant | no | When scored. |
| ReportSummary | inquiryId | String | no | Owning inquiry. |
| | hypothesis | String | no | Final hypothesis statement. |
| | supportingClaims | List<String> | no | Claim ids backing the hypothesis. |
| | openQuestions | List<String> | no | Sub-questions still open at convergence (may be empty). |
| | summarisedAt | Instant | no | When recorded. |

## Entity state — `Blackboard` (BlackboardEntity)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| inquiryId | String | no | Owning inquiry. |
| question | String | no | Original research question. |
| status | BlackboardStatus | no | See enum. |
| sources | List<Source> | no | All committed sources. |
| claims | List<Claim> | no | All committed claims. |
| hypotheses | List<Hypothesis> | no | All committed hypotheses. |
| disputed | List<Challenge> | no | All raised challenges. |
| openSubQuestions | List<String> | no | Sub-questions still without supporting claims. |
| iterationCount | int | no | Number of completed shell ticks. |
| report | Optional<ReportSummary> | yes | Populated on `Converged`. |
| lastWriteAt | Optional<Instant> | yes | Updated on every accepted write. |
| createdAt | Instant | no | When `BlackboardEntity` was created. |

## Entity state — `Inquiry` (InquiryEntity)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| inquiryId | String | no | Unique id. |
| question | String | no | Original question. |
| submittedBy | String | no | Submitter id. |
| status | InquiryStatus | no | See enum. |
| reportSummary | Optional<String> | yes | One-line summary on convergence. |
| createdAt | Instant | no | When `InquiryCreated` emitted. |
| completedAt | Optional<Instant> | yes | When inquiry closed. |

## Enums
- `BlackboardStatus`: OPEN, CONVERGED, CLOSED
- `InquiryStatus`: INTAKE, ACTIVE, CONVERGED, CLOSED

## Events — BlackboardEntity

| Event | Payload | Transition |
|---|---|---|
| SourceAdded | Source | appends to `sources`, updates `lastWriteAt` |
| ClaimAdded | Claim | appends to `claims`, updates `lastWriteAt` |
| HypothesisProposed | Hypothesis | appends to `hypotheses`, updates `lastWriteAt` |
| ChallengeRaised | Challenge | appends to `disputed`, updates `lastWriteAt` |
| SubQuestionOpened | text | appends to `openSubQuestions` |
| SubQuestionAnswered | text | removes from `openSubQuestions` |
| WriteRejected | ProposedWrite + failures | no state change (audit only) |
| IterationAdvanced | iterationCount | increments counter |
| Converged | ReportSummary | OPEN → CONVERGED, records `report` |
| Closed | reason, closedAt | (any) → CLOSED |

## Events — InquiryEntity

| Event | Payload | Transition |
|---|---|---|
| InquiryCreated | inquiryId, question, submittedBy, createdAt | → INTAKE |
| InquiryActivated | activatedAt | INTAKE → ACTIVE |
| InquiryConverged | summary | ACTIVE → CONVERGED |
| InquiryClosed | reason, closedAt | (any) → CLOSED |
| ReportRecorded | ReportSummary | sets `reportSummary` |

## Events — IntakeQueue

| Event | Payload |
|---|---|
| InquirySubmitted | inquiryId, question, submittedBy, submittedAt |

## Key-value state — SystemControl

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| halted | boolean | no | Whether the shell is frozen. |
| haltedReason | Optional<String> | yes | Operator-supplied reason. |
| haltedBy | Optional<String> | yes | Operator id. |
| haltedAt | Optional<Instant> | yes | When halted. |

## View row
`BlackboardRow` mirrors `Blackboard` but flattens the heavy lists into counts (`sourceCount`, `claimCount`, `hypothesisCount`, `disputedCount`) plus the most recent hypothesis statement and the most recent rejected-write reason. Every lifecycle field on the row is `Optional<T>` (Lesson 6). UI fetches the full blackboard only when the user expands a panel.
