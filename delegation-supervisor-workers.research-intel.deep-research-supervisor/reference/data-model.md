# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently -- the wire value is the raw value or `null`.

## ResearchReport (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `question` | `String` | no | The submitted research question |
| `status` | `ReportStatus` | no | Lifecycle state |
| `plan` | `Optional<DecompositionPlan>` | yes | Supervisor decomposition output; null until `PlanReady` |
| `subquerySummaries` | `Optional<List<SubquerySummary>>` | yes | Per-subquery summaries; null until first `SubquerySummaryAttached` |
| `synthesised` | `Optional<SynthesisedReport>` | yes | Merged report; null until `ReportSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1-5 citation-grounding score; null until `ReportEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `ReportView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record QuestionRequest(String question, String requestedBy) {}

record Subquery(String subqueryId, String queryText) {}
record DecompositionPlan(List<Subquery> subqueries) {}

record RawPassage(String passageId, String source, String text) {}
record PassageBundle(List<RawPassage> passages, Instant retrievedAt) {}

record ExtractedClaim(String claim, String sourcePassageId, String quote) {}
record ClaimsBundle(List<ExtractedClaim> claims, Instant extractedAt) {}

record SubquerySummary(String subqueryId, String queryText, String summary,
                       List<ExtractedClaim> supportingClaims, Instant summarisedAt) {}

record Citation(String citationId, String source, String quote) {}

record SynthesisedReport(String executiveSummary, List<SubquerySummary> subquerySummaries,
                         List<Citation> citationList, String guardrailVerdict,
                         Instant synthesisedAt) {}
```

## Status enum

```java
enum ReportStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

## Events

### ResearchReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) |
| `PlanReady` | Supervisor returns a `DecompositionPlan` |
| `SubquerySummaryAttached` | SummaryWorker returns a `SubquerySummary` for one subquery |
| `ReportSynthesised` | Supervisor synthesis passes the guardrail |
| `ReportDegraded` | One or more subqueries timed out; synthesised from partial input |
| `ReportBlocked` | Guardrail rejected the synthesised report (unverifiable citation) |
| `ReportEvalScored` | `CitationEvalSampler` recorded a 1-5 citation-grounding score |

### QuestionQueue

| Event | Trigger |
|---|---|
| `QuestionSubmitted` | `enqueueQuestion(question, requestedBy)` from endpoint or simulator |

Fields: `{ reportId, question, requestedBy, submittedAt }`.
