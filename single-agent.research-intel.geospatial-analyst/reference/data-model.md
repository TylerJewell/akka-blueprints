# Data model — earth-engine-geospatial

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Indicator` | `indicatorId` | `String` | no | Stable id supplied by the researcher. |
| | `name` | `String` | no | Human-readable indicator name. |
| | `datasetLayer` | `String` | no | Source raster product (e.g. `"MODIS/006/MOD13A2"`). |
| | `unit` | `String` | no | Physical unit (e.g. `"dimensionless"`, `"Kelvin"`, `"m"`). |
| `BoundingBox` | `swLat` | `double` | no | South-west corner latitude (degrees). |
| | `swLon` | `double` | no | South-west corner longitude (degrees). |
| | `neLat` | `double` | no | North-east corner latitude (degrees). |
| | `neLon` | `double` | no | North-east corner longitude (degrees). |
| `AnalysisQuery` | `analysisId` | `String` | no | UUID minted by `AnalysisEndpoint`. |
| | `queryLabel` | `String` | no | User-supplied description. |
| | `bounds` | `BoundingBox` | no | Spatial extent. |
| | `startDate` | `String` | no | ISO-8601 date string (e.g. `"2024-01-01"`). |
| | `endDate` | `String` | no | ISO-8601 date string (e.g. `"2024-12-31"`). |
| | `indicators` | `List<Indicator>` | no | Submitted list (1–N). |
| | `submittedBy` | `String` | no | Researcher identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `LayerStats` | `indicatorId` | `String` | no | Matches a submitted `Indicator.indicatorId`. |
| | `datasetLayer` | `String` | no | Source raster product, matches `Indicator.datasetLayer`. |
| | `meanValue` | `double` | no | Spatial mean over the bounding box. |
| | `minValue` | `double` | no | Spatial minimum. |
| | `maxValue` | `double` | no | Spatial maximum. |
| | `missingDataPct` | `double` | no | Percentage of pixels with no-data value (0.0–100.0). |
| | `unit` | `String` | no | Physical unit. |
| `DatasetSnapshot` | `layers` | `List<LayerStats>` | no | One entry per requested indicator. |
| | `pixelResolutionMeters` | `int` | no | Native pixel size (e.g. 30, 500). |
| | `totalPixels` | `long` | no | Total pixel count in the bounding box at this resolution. |
| `Finding` | `indicatorId` | `String` | no | MUST equal a submitted `indicatorId`. |
| | `severity` | `FindingSeverity` | no | Enum value. |
| | `datasetLayer` | `String` | no | MUST match a `LayerStats.datasetLayer`. |
| | `statistic` | `String` | no | Quantitative sentence citing dataset values. |
| | `anomalyDetected` | `boolean` | no | True if the finding flags a deviation. |
| | `anomalyConfidence` | `double` | no | 0.0–1.0; 0.0 when `anomalyDetected` is false. |
| | `recommendation` | `String` | no | Actionable verb-phrase. |
| `AnalysisReport` | `status` | `ReportStatus` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `findings` | `List<Finding>` | no | One entry per submitted indicator. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `EvaluationScorer` finished. |
| `Analysis` (entity state) | `analysisId` | `String` | no | — |
| | `query` | `Optional<AnalysisQuery>` | yes | Populated after `QuerySubmitted`. |
| | `dataset` | `Optional<DatasetSnapshot>` | yes | Populated after `DatasetReady`. |
| | `report` | `Optional<AnalysisReport>` | yes | Populated after `ReportRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `AnalysisStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Analysis` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`FindingSeverity`: `INFO`, `WATCH`, `ALERT`, `CRITICAL`.
`ReportStatus`: `NOMINAL`, `ANOMALY_DETECTED`, `INSUFFICIENT_DATA`.
`AnalysisStatus`: `SUBMITTED`, `DATASET_READY`, `ANALYSING`, `REPORT_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`AnalysisEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `query` | → SUBMITTED |
| `DatasetReady` | `dataset` | → DATASET_READY |
| `AnalysisStarted` | — | → ANALYSING |
| `ReportRecorded` | `report` | → REPORT_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `AnalysisFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Analysis.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AnalysisRow` mirrors `Analysis` minus `query.submittedBy` (private). The view declares ONE query: `getAllAnalyses: SELECT * AS analyses FROM analysis_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AnalysisTasks.java`)

```java
public final class AnalysisTasks {
  public static final Task<AnalysisReport> ANALYSE_REGION = Task
      .name("Analyse region")
      .description("Read the attached dataset snapshot and produce an AnalysisReport per submitted indicator")
      .resultConformsTo(AnalysisReport.class);

  private AnalysisTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
