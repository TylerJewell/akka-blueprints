# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## FinancialReport (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `query` | `String` | no | The submitted financial query |
| `status` | `ReportStatus` | no | Lifecycle state |
| `marketData` | `Optional<MarketDataBundle>` | yes | MarketResearcher output; null until `MarketDataAttached` |
| `planningAssessment` | `Optional<PlanningAssessment>` | yes | PortfolioPlanner output; null until `PlanningAttached` |
| `narrative` | `Optional<ReportNarrative>` | yes | ReportDrafter output; null until `NarrativeAttached` |
| `consolidated` | `Optional<ConsolidatedReport>` | yes | Merged report; null until `ReportPublished` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `ReportEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `FinancialView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record QueryRequest(String query, String requestedBy) {}
record WorkAssignment(String marketQuery, String planningQuestion, String narrativeFraming) {}
record MarketDataPoint(String ticker, String metric, String value, String source) {}
record MarketDataBundle(List<MarketDataPoint> dataPoints, Instant gatheredAt) {}
record PlanningAssessment(String riskProfile, List<String> allocationGuidance,
                          String rationale, Instant assessedAt) {}
record ReportNarrative(String narrativeText, List<String> keyMessages, Instant draftedAt) {}
record ConsolidatedReport(
    String executiveSummary,
    MarketDataBundle marketSection,
    PlanningAssessment planningSection,
    ReportNarrative narrative,
    String sanitizerVerdict,
    String guardrailVerdict,
    Instant synthesisedAt
) {}
```

## Status enum

```java
enum ReportStatus { DRAFTING, IN_REVIEW, PUBLISHED, DEGRADED, BLOCKED }
```

## Events

### FinancialReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) |
| `MarketDataAttached` | MarketResearcher returns a `MarketDataBundle` |
| `PlanningAttached` | PortfolioPlanner returns a `PlanningAssessment` |
| `NarrativeAttached` | ReportDrafter returns a `ReportNarrative` |
| `ReportPublished` | Coordinator synthesis passes sanitizer + guardrail |
| `ReportDegraded` | A worker timed out; synthesised from partial inputs |
| `ReportBlocked` | Guardrail rejected the consolidated report |
| `ReportEvalScored` | `EvalSampler` recorded a 1–5 quality score |

### QueryQueue

| Event | Trigger |
|---|---|
| `QuerySubmitted` | `enqueueQuery(query, requestedBy)` from endpoint or simulator |

Fields: `{ reportId, query, requestedBy, submittedAt }`.
