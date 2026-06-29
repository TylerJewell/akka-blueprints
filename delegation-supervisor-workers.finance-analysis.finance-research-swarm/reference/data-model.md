# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## StockReport (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `query` | `String` | no | The submitted company name or ticker query |
| `ticker` | `Optional<TickerResolution>` | yes | Resolved ticker; null until `TickerResolved` |
| `status` | `ReportStatus` | no | Lifecycle state |
| `priceSummary` | `Optional<PriceSummary>` | yes | PriceAnalyst output; null until `PriceAttached` |
| `newsSummary` | `Optional<NewsSummary>` | yes | NewsAnalyst output; null until `NewsAttached` |
| `fundamentals` | `Optional<FundamentalsSnapshot>` | yes | FundamentalsAnalyst output; null until `FundamentalsAttached` |
| `integrated` | `Optional<IntegratedReport>` | yes | Merged report; null until `ReportPublished` or `ReportPartial` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportPartial` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `EvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `StockReportView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record QueryRequest(String query, String requestedBy) {}
record QueryPlan(String tickerQuery, String priceQuestion,
                String newsQuestion, String fundamentalsQuestion) {}
record TickerResolution(String ticker, String exchange, String companyName) {}
record PricePoint(String date, double open, double close,
                  double high, double low, long volume) {}
record PriceSummary(String ticker, List<PricePoint> recentPrices,
                   double changePercent, String trend, Instant retrievedAt) {}
record NewsItem(String headline, String source, String publishedAt, String snippet) {}
record NewsSummary(String ticker, List<NewsItem> headlines,
                  String sentimentLabel, Instant retrievedAt) {}
record FundamentalsSnapshot(String ticker, double peRatio, double eps,
                            long marketCapUsd, double revenueGrowthPercent,
                            Instant retrievedAt) {}
record IntegratedReport(String summary, PriceSummary priceSummary,
                        NewsSummary newsSummary, FundamentalsSnapshot fundamentals,
                        String sanitizerVerdict, String guardrailVerdict,
                        Instant integratedAt) {}
```

## Status enum

```java
enum ReportStatus { RESOLVING, IN_PROGRESS, PUBLISHED, PARTIAL, BLOCKED }
```

## Events

### StockReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) |
| `TickerResolved` | TickerResolver returns a `TickerResolution` |
| `PriceAttached` | PriceAnalyst returns a `PriceSummary` |
| `NewsAttached` | NewsAnalyst returns a `NewsSummary` |
| `FundamentalsAttached` | FundamentalsAnalyst returns a `FundamentalsSnapshot` |
| `ReportPublished` | Integration, sanitizer, and guardrail all pass |
| `ReportPartial` | One or more workers timed out; integrated from partial inputs |
| `ReportBlocked` | Sanitizer or guardrail rejected the integrated report |
| `EvalScored` | `EvalSampler` recorded a 1–5 numeric accuracy score |

### RequestQueue

| Event | Trigger |
|---|---|
| `QuerySubmitted` | `enqueueQuery(query, requestedBy)` from endpoint or simulator |

Fields: `{ reportId, query, requestedBy, submittedAt }`.
