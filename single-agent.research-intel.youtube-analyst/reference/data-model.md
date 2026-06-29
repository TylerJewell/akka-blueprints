# Data model — youtube-analyst

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `VideoMetric` | `videoId` | `String` | no | Stable fixture identifier. |
| | `title` | `String` | no | Video title from fixture. |
| | `views` | `long` | no | Total view count over the period. |
| | `likes` | `long` | no | Total like count over the period. |
| | `comments` | `long` | no | Total comment count over the period. |
| | `engagementRate` | `double` | no | `(likes + comments) / views × 100`. |
| | `publishedAt` | `String` | no | ISO-8601 date string (`yyyy-MM-dd`). |
| `ChannelMetrics` | `channelId` | `String` | no | Fixture channel identifier. |
| | `channelHandle` | `String` | no | e.g., `@devtoolscreator`. |
| | `subscriberCount` | `long` | no | Subscriber count at period end. |
| | `subscriberDelta` | `long` | no | Change over the requested period. |
| | `totalViews` | `long` | no | Cumulative views over the period. |
| | `viewsDelta` | `long` | no | Views change over the period. |
| | `period` | `String` | no | `"28d"` / `"90d"` / `"all"`. |
| | `topVideos` | `List<VideoMetric>` | no | Up to 10 videos ranked by engagementRate. |
| | `fetchedAt` | `Instant` | no | When `MetricsFetcher` built the object. |
| `AnalysisRequest` | `analysisId` | `String` | no | UUID minted by `AnalysisEndpoint`. |
| | `channelHandle` | `String` | no | User-supplied channel handle. |
| | `period` | `String` | no | Requested analysis period. |
| | `focusArea` | `FocusArea` | no | Enum value. |
| | `requestedBy` | `String` | no | User identifier. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `TopVideo` | `videoId` | `String` | no | MUST match a `VideoMetric.videoId` in the attachment. |
| | `title` | `String` | no | MUST match the fixture title. |
| | `views` | `long` | no | Carried from fixture. |
| | `engagementRate` | `double` | no | Carried from fixture; never zero. |
| | `trend` | `TrendDirection` | no | Relative to channel average. |
| `Recommendation` | `metricRef` | `String` | no | One of: `subscriberCount`, `subscriberDelta`, `totalViews`, `viewsDelta`, `engagementRate`. |
| | `action` | `String` | no | Actionable verb-phrase. |
| `ChannelReport` | `tier` | `PerformanceTier` | no | Enum value. |
| | `summary` | `String` | no | 2–4 sentences with at least one numeric figure. |
| | `topVideos` | `List<TopVideo>` | no | 3–5 entries. |
| | `recommendations` | `List<Recommendation>` | no | 2–4 entries; never empty. |
| | `reportedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `AnalysisEvaluator` finished. |
| `ChannelAnalysis` (entity state) | `analysisId` | `String` | no | — |
| | `request` | `Optional<AnalysisRequest>` | yes | Populated after `AnalysisRequested`. |
| | `metrics` | `Optional<ChannelMetrics>` | yes | Populated after `MetricsFetched`. |
| | `report` | `Optional<ChannelReport>` | yes | Populated after `ReportRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `AnalysisStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AnalysisRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ChannelAnalysis` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`FocusArea`: `GROWTH`, `ENGAGEMENT`, `AUDIENCE`, `ALL`.
`TrendDirection`: `UP`, `FLAT`, `DOWN`.
`PerformanceTier`: `GROWING`, `STEADY`, `DECLINING`.
`AnalysisStatus`: `REQUESTED`, `METRICS_FETCHED`, `ANALYSING`, `REPORT_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`ChannelEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AnalysisRequested` | `request` | → REQUESTED |
| `MetricsFetched` | `metrics` | → METRICS_FETCHED |
| `AnalysisStarted` | — | → ANALYSING |
| `ReportRecorded` | `report` | → REPORT_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `AnalysisFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ChannelAnalysis.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AnalysisRow` mirrors `ChannelAnalysis`. The view declares ONE query: `getAllAnalyses: SELECT * AS analyses FROM analysis_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AnalysisTasks.java`)

```java
public final class AnalysisTasks {
  public static final Task<ChannelReport> ANALYSE_CHANNEL = Task
      .name("Analyse channel")
      .description("Read the attached channel metrics and produce a ChannelReport with tier classification, top-video table, and recommendations")
      .resultConformsTo(ChannelReport.class);

  private AnalysisTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
