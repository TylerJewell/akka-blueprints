# Data model — hvac-analytics

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TelemetryPoint` | `metric` | `String` | no | Metric name, e.g. `"supply-air-temp"`, `"airflow-cfm"`. |
| | `value` | `double` | no | Numeric reading. |
| | `unit` | `String` | no | Unit of measure, e.g. `"°F"`, `"cfm"`, `"kWh"`. |
| | `zone` | `String` | no | Zone label, e.g. `"Zone-A"`. No equipment serial numbers. |
| | `recordedAt` | `Instant` | no | When the reading was recorded. |
| `TelemetrySnapshot` | `points` | `List<TelemetryPoint>` | no | All telemetry for the requested scope. |
| | `assembledAt` | `Instant` | no | When `TelemetryStore` built this snapshot. |
| | `zoneScope` | `String` | no | Zone(s) included, e.g. `"Zone-A"` or `"Zone-A,Zone-B"`. |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `question` | `String` | no | The engineer's natural-language question. |
| | `zoneScope` | `String` | no | Zone(s) the question covers. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `DataPoint` | `metric` | `String` | no | Metric name from the snapshot. |
| | `value` | `double` | no | Numeric value. |
| | `unit` | `String` | no | Unit of measure. |
| | `zone` | `String` | no | Zone label. |
| | `recordedAt` | `Instant` | no | When this reading was recorded. |
| | `significance` | `String` | no | Why this point supports the answer. Must not be empty. |
| `AnalyticsAnswer` | `assessment` | `String` | no | Plain-language 1–3 sentence paragraph. |
| | `dataPoints` | `List<DataPoint>` | no | Supporting evidence. Must not be empty. |
| | `trend` | `TrendClassification` | no | Enum value. |
| | `recommendedAction` | `String` | no | Begins with an actionable verb. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `AnswerQualityScorer` finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QueryInitiated`. |
| | `snapshot` | `Optional<TelemetrySnapshot>` | yes | Populated after `SnapshotAttached`. |
| | `answer` | `Optional<AnalyticsAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryInitiated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TrendClassification`: `IMPROVING`, `STABLE`, `DEGRADING`, `UNKNOWN`.
`QueryStatus`: `INITIATED`, `SNAPSHOT_READY`, `ANALYSING`, `ANSWER_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryInitiated` | `request` | → INITIATED |
| `SnapshotAttached` | `snapshot` | → SNAPSHOT_READY |
| `AnalysisStarted` | — | → ANALYSING |
| `AnswerRecorded` | `answer` | → ANSWER_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = INITIATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query`. The view does not strip any fields — the telemetry snapshot already has equipment serial numbers removed by `TelemetryStore` before it reaches the entity.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<AnalyticsAnswer> ANALYSE_TELEMETRY = Task
      .name("Analyse telemetry")
      .description("Read the attached telemetry snapshot and produce an AnalyticsAnswer for the submitted question")
      .resultConformsTo(AnalyticsAnswer.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
