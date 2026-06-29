# Data model — google-trends-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TrendQuery` | `requestId` | `String` | no | UUID minted by `TrendEndpoint`. |
| | `region` | `String` | no | BCP-47 region code (e.g. `"US"`, `"DE"`, `"JP"`). |
| | `timeWindow` | `TimeWindow` | no | Enum value. |
| | `category` | `String` | no | Interest category (e.g. `"Technology"`, `"All"`). |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `RawTrendPayload` | `region` | `String` | no | Copied from `TrendQuery`. |
| | `timeWindow` | `TimeWindow` | no | Copied from `TrendQuery`. |
| | `category` | `String` | no | Copied from `TrendQuery`. |
| | `topics` | `List<RawTopic>` | no | Raw data rows from the seed dataset (1–N). |
| | `fetchedAt` | `Instant` | no | When `TrendDataFetcher` assembled the payload. |
| `RawTopic` | `topicName` | `String` | no | Exact topic name from the seed row. |
| | `searchVolumeIndex` | `int` | no | Relative search volume, 0–100 scale. |
| | `breakout` | `boolean` | no | True if growth exceeded 5 000% in the time window. |
| | `relatedQueries` | `List<String>` | no | Up to 5 related queries from the seed row. |
| `TrendingTopic` | `rank` | `int` | no | 1-based position; strictly increasing, no gaps. |
| | `topicName` | `String` | no | Exact `topicName` from the corresponding `RawTopic`. |
| | `searchVolumeIndex` | `int` | no | Exact `searchVolumeIndex` from the corresponding `RawTopic`. |
| | `breakout` | `boolean` | no | Exact `breakout` from the corresponding `RawTopic`. |
| | `rationale` | `String` | no | Agent's ≥ 10-word explanation. |
| `RelatedCluster` | `anchorTerm` | `String` | no | MUST equal a `topicName` in the report's `topics` list. |
| | `relatedQueries` | `List<String>` | no | Top 3 from the seed row's `relatedQueries`. |
| `TrendReport` | `region` | `String` | no | Copied from `TrendQuery`. |
| | `timeWindow` | `TimeWindow` | no | Copied from `TrendQuery`. |
| | `category` | `String` | no | Copied from `TrendQuery`. |
| | `topics` | `List<TrendingTopic>` | no | Ranked 1..N; one per `RawTopic` in the payload. |
| | `breakoutSummary` | `String` | no | Agent's ≥ 20-word breakout signal paragraph. |
| | `clusters` | `List<RelatedCluster>` | no | One per topic with non-empty `relatedQueries`. |
| | `reportedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `ReportEvaluationScorer` finished. |
| `TrendRequest` (entity state) | `requestId` | `String` | no | — |
| | `query` | `Optional<TrendQuery>` | yes | Populated after `TrendQuerySubmitted`. |
| | `rawPayload` | `Optional<RawTrendPayload>` | yes | Populated after `TrendDataReady`. |
| | `report` | `Optional<TrendReport>` | yes | Populated after `ReportRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `TrendRequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TrendQuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `TrendRequest` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TimeWindow`: `PAST_DAY`, `PAST_WEEK`, `PAST_MONTH`, `PAST_90_DAYS`.

`TrendRequestStatus`: `SUBMITTED`, `DATA_READY`, `SYNTHESIZING`, `REPORT_READY`, `EVALUATED`, `FAILED`.

## Events (`TrendRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TrendQuerySubmitted` | `query` | → SUBMITTED |
| `TrendDataReady` | `rawPayload` | → DATA_READY |
| `SynthesisStarted` | — | → SYNTHESIZING |
| `ReportRecorded` | `report` | → REPORT_READY |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `TrendRequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `TrendRequest.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`TrendReportRow` mirrors `TrendRequest` minus the full `RawTrendPayload.topics` body — the view stores payload metadata only (`region`, `timeWindow`, `category`, `topicCount`). The full raw payload is available via `GET /api/trends/{id}`, which reads directly from the entity's event log.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM trend_report_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`TrendTasks.java`)

```java
public final class TrendTasks {
  public static final Task<TrendReport> SYNTHESIZE_TRENDS = Task
      .name("Synthesize trends")
      .description("Read the attached trend-data payload and produce a TrendReport with ranked topics, breakout signals, and related query clusters")
      .resultConformsTo(TrendReport.class);

  private TrendTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
