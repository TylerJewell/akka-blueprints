# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## ContractReview (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reviewId` | `String` | no | UUID, also the workflow id |
| `contractRef` | `String` | no | Caller-supplied contract reference name |
| `contractText` | `String` | no | Full contract text as submitted |
| `submittedBy` | `String` | no | Identifier of the submitting user |
| `status` | `ReviewStatus` | no | Lifecycle state |
| `clauseSummary` | `Optional<ClauseSummary>` | yes | ClauseAnalyst output; null until `ClauseSummaryAttached` |
| `riskReport` | `Optional<RiskReport>` | yes | RiskScorer output; null until `RiskReportAttached` |
| `redlines` | `Optional<RedlineSet>` | yes | Redliner output; null until `RedlineSetAttached` |
| `reviewPackage` | `Optional<ReviewPackage>` | yes | Consolidated package; null until `ReviewConsolidated` |
| `failureReason` | `Optional<String>` | yes | Set on `ReviewBlocked`, `ReviewDegraded`, or `ReviewRejectedBySanitizer` |
| `lawyerNote` | `Optional<String>` | yes | Set on `ReviewApproved` or `ReviewRejectedByLawyer` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `ReviewEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Review creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `ReviewView` row type (`ContractReviewRow`) mirrors this with `Optional<T>` on every nullable field. Heavy nested payloads (`contractText`) may be omitted from the row type; the full text is available from `GET /api/reviews/{id}`.

## Supporting records

```java
record ContractSubmission(String contractRef, String contractText, String submittedBy) {}

record ClauseEntry(String clauseId, String clauseType, String excerpt, String summary) {}
record ClauseSummary(List<ClauseEntry> clauses, Instant analysedAt) {}

record RiskFlag(String clauseId, RiskLevel riskLevel, String rationale) {}
record RiskReport(List<RiskFlag> flags, RiskLevel overallRisk, Instant scoredAt) {}

record RedlineSuggestion(String clauseId, String originalText, String proposedText, String justification) {}
record RedlineSet(List<RedlineSuggestion> suggestions, Instant draftedAt) {}

record ReviewPlan(String clauseExtractionTask, String riskScoringTask, String redlineTask) {}

record ReviewPackage(ClauseSummary clauseSummary, RiskReport riskReport,
                     RedlineSet redlines, String guardrailVerdict, Instant consolidatedAt) {}
```

## Status enum

```java
enum ReviewStatus { QUEUED, IN_REVIEW, AWAITING_APPROVAL, APPROVED, REJECTED, DEGRADED, BLOCKED }
enum RiskLevel { LOW, MEDIUM, HIGH, CRITICAL }
```

## Events

### ContractReviewEntity

| Event | Trigger |
|---|---|
| `ReviewCreated` | Workflow creates the review (`createReview`) |
| `ReviewRejectedBySanitizer` | Legal-sector sanitizer matches a prohibited pattern |
| `ClauseSummaryAttached` | ClauseAnalyst returns a `ClauseSummary` |
| `RiskReportAttached` | RiskScorer returns a `RiskReport` |
| `RedlineSetAttached` | Redliner returns a `RedlineSet` |
| `ReviewConsolidated` | Supervisor consolidation passes the guardrail; moves to `AWAITING_APPROVAL` |
| `ReviewDegraded` | A worker timed out or the approval window lapsed; synthesised from partial input |
| `ReviewBlocked` | Guardrail rejected the consolidated package |
| `ReviewApproved` | Lawyer approved the review package (`approve`) |
| `ReviewRejectedByLawyer` | Lawyer rejected the review package (`rejectByLawyer`) |
| `ReviewEvalScored` | `ReviewSampler` recorded a 1–5 quality score |

### ContractQueue

| Event | Trigger |
|---|---|
| `ContractSubmitted` | `enqueueContract(contractRef, contractText, submittedBy)` from endpoint or simulator |

Fields: `{ reviewId, contractRef, submittedBy, submittedAt }`.
