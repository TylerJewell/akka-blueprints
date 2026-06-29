# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## PortfolioReport (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `portfolioId` | `String` | no | Caller-supplied portfolio identifier |
| `sector` | `String` | no | Normalized sector tag from `SectorRegistry` |
| `status` | `ReportStatus` | no | Lifecycle state |
| `holdingsAssessment` | `Optional<PositionAssessment>` | yes | HoldingsAnalyst output; null until `HoldingsAssessed` |
| `marketContext` | `Optional<MarketContext>` | yes | MarketContextAgent output; null until `MarketContextAttached` |
| `consolidated` | `Optional<ConsolidatedReport>` | yes | Merged report; null until `ReportConsolidated` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `ReportEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `PortfolioView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record Holding(String ticker, String name, double weight, double marketValue) {}
record HoldingsRequest(String portfolioId, String sector, List<Holding> holdings, String submittedBy) {}
record AnalysisPlan(String positionQuery, String marketQuestion) {}
record PositionNote(String ticker, String observation, String riskLevel) {}
record PositionAssessment(List<PositionNote> notes, String sectorExposure, Instant assessedAt) {}
record MarketContext(String macroSummary, List<String> sectorHeadwinds, List<String> sectorTailwinds, Instant gatheredAt) {}
record ConsolidatedReport(String executive, PositionAssessment holdingsAssessment,
                          MarketContext marketContext,
                          String guardrailVerdict, Instant consolidatedAt) {}
```

## Status enum

```java
enum ReportStatus { PLANNING, IN_PROGRESS, CONSOLIDATED, DEGRADED, BLOCKED }
```

## Events

### PortfolioReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) after sector check passes |
| `HoldingsAssessed` | HoldingsAnalyst returns a `PositionAssessment` |
| `MarketContextAttached` | MarketContextAgent returns a `MarketContext` |
| `ReportConsolidated` | Coordinator consolidation passes the guardrail |
| `ReportDegraded` | A worker timed out; consolidated from partial input |
| `ReportBlocked` | Sector sanitizer rejected the sector OR guardrail rejected the consolidated report |
| `ReportEvalScored` | `EvalSampler` recorded a 1–5 score |

### HoldingsQueue

| Event | Trigger |
|---|---|
| `HoldingsSubmitted` | `enqueueHoldings(portfolioId, sector, holdings, submittedBy)` from endpoint or simulator |

Fields: `{ reportId, portfolioId, sector, holdings, submittedBy, submittedAt }`.

## SectorRegistry (domain utility)

`SectorRegistry.java` holds two static collections:

- `PERMITTED_SECTORS` — canonical normalized sector strings (e.g., `"technology"`, `"healthcare"`, `"financials"`, `"energy"`, `"consumer-discretionary"`, `"industrials"`, `"utilities"`, `"materials"`, `"real-estate"`, `"communication-services"`, `"consumer-staples"`).
- `PROHIBITED_SECTORS` — sectors blocked at the gate (e.g., `"weapons"`, `"tobacco"`, `"gambling"`).

`SectorRegistry.normalize(String raw)` lowercases and trims the input; returns the matched canonical string or throws `ProhibitedSectorException` / `UnknownSectorException` accordingly.
