# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## StockReport (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `ticker` | `String` | no | The submitted stock ticker symbol |
| `status` | `ReportStatus` | no | Lifecycle state |
| `financials` | `Optional<FinancialMetrics>` | yes | FinancialsAgent output; null until `FinancialsAttached` |
| `news` | `Optional<NewsSummary>` | yes | NewsAgent output; null until `NewsAttached` |
| `ratios` | `Optional<RatioSet>` | yes | RatioAgent output; null until `RatiosAttached` |
| `recommendation` | `Optional<StockRecommendation>` | yes | Synthesised output; null until `ReportCompleted` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `ReportEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `StockReportView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TickerRequest(String ticker, String requestedBy) {}

record AnalysisSubtasks(String financialsQuery, String newsQuery, String ratiosQuery) {}

record FinancialMetrics(
    String ticker,
    Optional<Double> revenueGrowthPct,
    Optional<Double> epsLatest,
    Optional<Double> operatingMarginPct,
    Instant extractedAt
) {}

record NewsItem(String headline, String source, String sentiment) {}
record NewsSummary(
    String ticker,
    List<NewsItem> items,
    String overallSentiment,
    Instant summarisedAt
) {}

record RatioSet(
    String ticker,
    Optional<Double> peRatio,
    Optional<Double> pbRatio,
    Optional<Double> roe,
    Optional<Double> debtEquityRatio,
    Instant computedAt
) {}

record StockRecommendation(
    String ticker,
    String summary,
    FinancialMetrics financialHighlights,
    NewsSummary newsSentiment,
    RatioSet ratioAnalysis,
    RecommendationStance stance,
    boolean disclaimerPresent,
    String sanitizerVerdict,
    Instant synthesisedAt
) {}
```

## Status enums

```java
enum ReportStatus { QUEUED, ANALYSING, COMPLETE, DEGRADED, BLOCKED }
enum RecommendationStance { POSITIVE, NEUTRAL, CAUTIOUS, INSUFFICIENT_DATA }
```

## Events

### StockReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) |
| `FinancialsAttached` | FinancialsAgent returns `FinancialMetrics` |
| `NewsAttached` | NewsAgent returns `NewsSummary` |
| `RatiosAttached` | RatioAgent returns `RatioSet` |
| `ReportCompleted` | Coordinator synthesis passes guardrail and sanitizer |
| `ReportDegraded` | One or more workers timed out; synthesised from partial input |
| `ReportBlocked` | Guardrail or sanitizer rejected the synthesised recommendation |
| `ReportEvalScored` | `EvalSampler` recorded a 1–5 factuality score |

### TickerQueue

| Event | Trigger |
|---|---|
| `TickerSubmitted` | `enqueueTicker(ticker, requestedBy)` from endpoint or simulator |

Fields: `{ reportId, ticker, requestedBy, submittedAt }`.
