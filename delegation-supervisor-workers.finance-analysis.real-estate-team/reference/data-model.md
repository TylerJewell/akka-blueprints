# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## DealRecord (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `dealId` | `String` | no | UUID, also the workflow id |
| `address` | `String` | no | Street address of the subject property |
| `propertyType` | `String` | no | Property category (multifamily, single-family, commercial, etc.) |
| `askingPriceDollars` | `double` | no | Listed asking price in USD |
| `status` | `DealStatus` | no | Lifecycle state |
| `marketReport` | `Optional<MarketReport>` | yes | MarketSpecialist output; null until `MarketReportAttached` |
| `financialModel` | `Optional<FinancialModel>` | yes | FinanceSpecialist output; null until `FinancialModelAttached` |
| `recommendation` | `Optional<InvestmentRecommendation>` | yes | Merged recommendation; null until `DealRecommended` |
| `failureReason` | `Optional<String>` | yes | Set on `DealBlocked`, `DealDegraded`, or `DealRejected` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `RecommendationScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Deal creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `DealView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record DealSubmission(String address, String propertyType,
                      double askingPriceDollars, String submittedBy) {}

record EvaluationPlan(String marketQuery, String financialQuestion) {}

record Comparable(String address, double salePriceDollars, String saleDate, double sqft) {}

record MarketReport(List<Comparable> comparables, String neighborhoodTrend,
                    double estimatedMarketValueDollars, Instant gatheredAt) {}

record FinancialModel(double grossRentalIncome, double operatingExpenses,
                      double netOperatingIncome, double capRate,
                      double cashOnCashReturn, double debtServiceCoverageRatio,
                      Instant modelledAt) {}

record InvestmentRecommendation(String summary, MarketReport marketReport,
                                FinancialModel financialModel,
                                String recommendationVerdict,
                                String guardrailVerdict,
                                Instant synthesisedAt) {}
```

## Status enum

```java
enum DealStatus { SCOPING, EVALUATING, RECOMMENDED, DEGRADED, BLOCKED, REJECTED }
```

## Events

### DealEntity

| Event | Trigger |
|---|---|
| `DealCreated` | Workflow creates the deal after sanitizer passes (`createDeal`) |
| `DealRejected` | Sanitizer matched an excluded property type (`rejectDeal`) |
| `MarketReportAttached` | MarketSpecialist returns a `MarketReport` |
| `FinancialModelAttached` | FinanceSpecialist returns a `FinancialModel` |
| `DealRecommended` | Coordinator synthesis passes the guardrail |
| `DealDegraded` | A specialist timed out; synthesised from partial input |
| `DealBlocked` | Guardrail rejected the synthesised recommendation |
| `RecommendationScored` | `RecommendationEvalSampler` recorded a 1–5 quality score |

### DealQueue

| Event | Trigger |
|---|---|
| `DealSubmitted` | `submitDeal(address, propertyType, askingPriceDollars, submittedBy)` from endpoint or simulator |

Fields: `{ dealId, address, propertyType, askingPriceDollars, submittedBy, submittedAt }`.
