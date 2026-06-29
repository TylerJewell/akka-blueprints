# Data model — time-series-forecasting (java)

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SeriesConfig` | `horizonDays` | `int` | no | Number of days (or weeks) to forecast ahead. |
| | `confidenceLevel` | `double` | no | Interval confidence (0.80 or 0.95). |
| | `granularity` | `String` | no | `"daily"` or `"weekly"`. |
| `SeriesPoint` | `date` | `LocalDate` | no | ISO-8601 date of this observation. |
| | `value` | `double` | no | Numeric observation value. |
| `SeriesSubmission` | `forecastId` | `String` | no | UUID minted by `ForecastEndpoint`. |
| | `seriesName` | `String` | no | User-supplied label. |
| | `historicalData` | `List<SeriesPoint>` | no | Raw input series. Audit-only after validation. |
| | `config` | `SeriesConfig` | no | Forecast configuration. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SeriesQuality` | `rowCount` | `int` | no | Number of rows in the series. |
| | `rangeStart` | `LocalDate` | no | Earliest date in the series. |
| | `rangeEnd` | `LocalDate` | no | Latest date in the series. |
| | `missingValueCount` | `int` | no | Count of rows with null or non-numeric value. |
| | `isStationary` | `boolean` | no | ADF approximation result. |
| | `qualitySummary` | `String` | no | One-line human-readable summary. |
| `HorizonStep` | `forecastDate` | `LocalDate` | no | The calendar date being forecast. |
| | `pointForecast` | `double` | no | Central estimate. |
| | `lowerBound` | `double` | no | Must satisfy `lowerBound <= pointForecast`. |
| | `upperBound` | `double` | no | Must satisfy `pointForecast <= upperBound`. |
| | `stepQualityScore` | `int` | no | 1–5; typically decreases for distant steps. |
| `ForecastResult` | `steps` | `List<HorizonStep>` | no | One entry per horizon day (or week). |
| | `summary` | `String` | no | 2–4 sentences from the agent. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `DriftEval` | `driftStatus` | `DriftStatus` | no | Enum value. |
| | `mape` | `double` | no | Mean absolute percentage error (0.0–1.0+). |
| | `rationale` | `String` | no | One sentence explaining the status. |
| | `evaluatedAt` | `Instant` | no | When `DriftEvaluator` finished. |
| `Forecast` (entity state) | `forecastId` | `String` | no | — |
| | `submission` | `Optional<SeriesSubmission>` | yes | Populated after `SeriesSubmitted`. |
| | `quality` | `Optional<SeriesQuality>` | yes | Populated after `SeriesValidated`. |
| | `result` | `Optional<ForecastResult>` | yes | Populated after `ForecastCompleted`. |
| | `driftEval` | `Optional<DriftEval>` | yes | Populated after `DriftEvaluated`. |
| | `status` | `ForecastStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SeriesSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Forecast` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DriftStatus`: `STABLE`, `WATCH`, `ALERT`.
`ForecastStatus`: `SUBMITTED`, `VALIDATED`, `FORECASTING`, `FORECAST_COMPLETED`, `DRIFT_EVALUATED`, `FAILED`.

## Events (`ForecastEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SeriesSubmitted` | `submission` | → SUBMITTED |
| `SeriesValidated` | `quality` | → VALIDATED |
| `ForecastStarted` | — | → FORECASTING |
| `ForecastCompleted` | `result` | → FORECAST_COMPLETED |
| `DriftEvaluated` | `driftEval` | → DRIFT_EVALUATED (terminal happy) |
| `ForecastFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Forecast.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ForecastRow` mirrors `Forecast` minus `submission.historicalData` (the audit log keeps the raw series). The UI fetches the full series on demand via `GET /api/forecasts/{id}` and reads `submission.historicalData` from the JSON.

The view declares ONE query: `getAllForecasts: SELECT * AS forecasts FROM forecast_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ForecastTasks.java`)

```java
public final class ForecastTasks {
  public static final Task<ForecastResult> FORECAST_SERIES = Task
      .name("Forecast series")
      .description("Read the attached historical series and produce a ForecastResult with one HorizonStep per requested horizon day")
      .resultConformsTo(ForecastResult.class);

  private ForecastTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
