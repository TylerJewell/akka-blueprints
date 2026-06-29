# Data model — currency-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ConversionRequest` | `conversionId` | `String` | no | UUID minted by `ConversionEndpoint`. |
| | `fromCurrency` | `String` | no | ISO 4217 source currency code. |
| | `toCurrency` | `String` | no | ISO 4217 target currency code. |
| | `amount` | `BigDecimal` | no | Amount to convert. |
| | `snapshotLabel` | `String` | no | `spot` / `morning-fix` / `end-of-day`. |
| | `requestedBy` | `String` | no | User identifier. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `RateSnapshot` | `label` | `String` | no | Matches `snapshotLabel` on the request. |
| | `fromCurrency` | `String` | no | ISO 4217 source currency. |
| | `toCurrency` | `String` | no | ISO 4217 target currency. |
| | `rate` | `BigDecimal` | no | Exchange rate (1 unit of `fromCurrency` in `toCurrency`). |
| | `capturedAt` | `Instant` | no | When the rate was captured. Used by `FreshnessScorer`. |
| | `source` | `String` | no | e.g. `ecb-fix`, `spot-feed`, `seed-file`. |
| `ConversionResult` | `convertedAmount` | `BigDecimal` | no | `amount × rate`, 4 decimal places. |
| | `rateApplied` | `BigDecimal` | no | Rate value from the snapshot. |
| | `fromCurrency` | `String` | no | Echoes the request. |
| | `toCurrency` | `String` | no | Echoes the request. |
| | `confidenceNote` | `String` | no | 1 sentence on rate-data quality. |
| | `marketContext` | `String` | no | 1–2 sentences of market context. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `FreshnessEval` | `score` | `int` | no | 1–5. Higher = fresher rate data. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `evaluatedAt` | `Instant` | no | When `FreshnessScorer` finished. |
| `Conversion` (entity state) | `conversionId` | `String` | no | — |
| | `request` | `Optional<ConversionRequest>` | yes | Populated after `ConversionRequested`. |
| | `rateSnapshot` | `Optional<RateSnapshot>` | yes | Populated after `RateAttached`. |
| | `result` | `Optional<ConversionResult>` | yes | Populated after `ResultRecorded`. |
| | `eval` | `Optional<FreshnessEval>` | yes | Populated after `FreshnessScored`. |
| | `status` | `ConversionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ConversionRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Conversion` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ConversionStatus`: `REQUESTED`, `RATE_ATTACHED`, `CONVERTING`, `RESULT_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`ConversionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversionRequested` | `request` | → REQUESTED |
| `RateAttached` | `rateSnapshot` | → RATE_ATTACHED |
| `ConversionStarted` | — | → CONVERTING |
| `ResultRecorded` | `result` | → RESULT_RECORDED |
| `FreshnessScored` | `eval` | → EVALUATED (terminal happy) |
| `ConversionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Conversion.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversionRow` mirrors `Conversion` fully (there is no sensitive field analogous to `rawDocument` in the docreview template — rate data and results are non-sensitive by default). The UI fetches the full detail on-demand via `GET /api/conversions/{id}`.

The view declares ONE query: `getAllConversions: SELECT * AS conversions FROM conversion_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ConversionTasks.java`)

```java
public final class ConversionTasks {
  public static final Task<ConversionResult> CONVERT_CURRENCY = Task
      .name("Convert currency")
      .description("Read the attached rate snapshot and return a ConversionResult for the given currency pair and amount")
      .resultConformsTo(ConversionResult.class);

  private ConversionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
