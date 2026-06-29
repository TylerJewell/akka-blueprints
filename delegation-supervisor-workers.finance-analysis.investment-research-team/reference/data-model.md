# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## ResearchReport (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `ticker` | `String` | no | The submitted ticker symbol |
| `question` | `String` | no | The submitted research question |
| `status` | `ReportStatus` | no | Lifecycle state |
| `fundamentals` | `Optional<FundamentalsBundle>` | yes | EquityAnalyst output; null until `FundamentalsAttached` |
| `sentiment` | `Optional<SentimentBundle>` | yes | MarketScout output; null until `SentimentAttached` |
| `synthesised` | `Optional<SynthesisedReport>` | yes | Merged report; null until `ReportPublished` or `ReportDegraded` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `ReportEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line numeric-claim quality justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `ReportView` row type `ResearchReportRow` mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record ResearchRequest(String ticker, String question, String requestedBy) {}
record ResearchPlan(String fundamentalQuery, String sentimentQuery) {}
record FundamentalFact(String metric, String value, String period, String source) {}
record FundamentalsBundle(List<FundamentalFact> facts, Instant gatheredAt) {}
record SentimentSignal(String headline, String direction, String source) {}
record SentimentBundle(String overallSentiment, List<SentimentSignal> signals, Instant gatheredAt) {}
record SynthesisedReport(String summary, FundamentalsBundle fundamentals,
                         SentimentBundle sentiment, String sanitizerVerdict, Instant synthesisedAt) {}
```

## Status enum

```java
enum ReportStatus { PLANNING, IN_PROGRESS, PUBLISHED, DEGRADED, BLOCKED }
```

## Events

### ResearchReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) |
| `FundamentalsAttached` | EquityAnalyst returns a `FundamentalsBundle` |
| `SentimentAttached` | MarketScout returns a `SentimentBundle` |
| `ReportPublished` | Coordinator synthesis passes the sector sanitizer |
| `ReportDegraded` | A worker timed out; synthesised from partial input |
| `ReportBlocked` | Sector sanitizer rejected the synthesised report |
| `ReportEvalScored` | `EvalSampler` recorded a 1–5 numeric-claim quality score |

### RequestQueue

| Event | Trigger |
|---|---|
| `ResearchSubmitted` | `enqueueRequest(ticker, question, requestedBy)` from endpoint or simulator |

Fields: `{ reportId, ticker, question, requestedBy, submittedAt }`.
