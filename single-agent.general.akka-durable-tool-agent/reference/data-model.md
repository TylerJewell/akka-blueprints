# Data model — durable-weather-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WeatherQuery` | `jobId` | `String` | no | UUID minted by `JobEndpoint`. |
| | `queryText` | `String` | no | User's free-text weather question. |
| | `city` | `String` | yes | Resolved city name; null until the agent resolves it. |
| | `mode` | `InvocationMode` | no | Enum: TRIGGER_AGENT or CALL_AGENT. |
| | `submittedBy` | `String` | no | Caller identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallRecord` | `toolName` | `String` | no | e.g. `"get_current_weather"`. |
| | `argumentsSummary` | `String` | no | Human-readable argument summary. |
| | `status` | `ToolCallStatus` | no | Enum: EXECUTED, BLOCKED, FAILED. |
| | `rawResponseSnippet` | `Optional<String>` | yes | Truncated raw response; empty on BLOCKED. |
| | `guardRejectionReason` | `Optional<String>` | yes | Rejection message; empty on EXECUTED. |
| | `calledAt` | `Instant` | no | When the activity was invoked. |
| `WeatherConditions` | `city` | `String` | no | Resolved city name. |
| | `country` | `String` | no | ISO-3166 alpha-2 country code. |
| | `temperatureCelsius` | `double` | no | Current temperature. |
| | `conditionLabel` | `String` | no | e.g. `"Partly cloudy"`. |
| | `humidityPercent` | `int` | no | 0–100. |
| | `windSpeedKph` | `double` | no | Wind speed. |
| | `observedAt` | `Instant` | no | Observation timestamp from the API. |
| `ForecastDay` | `date` | `String` | no | ISO-8601 date string, e.g. `"2026-06-29"`. |
| | `highCelsius` | `double` | no | Forecast high temperature. |
| | `lowCelsius` | `double` | no | Forecast low temperature. |
| | `conditionLabel` | `String` | no | e.g. `"Rain showers"`. |
| | `precipitationChancePercent` | `int` | no | 0–100. |
| `WeatherReport` | `jobId` | `String` | no | Echoed from the job. |
| | `current` | `WeatherConditions` | no | Current conditions. |
| | `forecast` | `List<ForecastDay>` | no | Ordered by date ascending. |
| | `activeAlerts` | `List<String>` | no | Empty list if none or not checked. |
| | `narrativeSummary` | `String` | no | 2–4 sentences. |
| | `toolCallLog` | `List<ToolCallRecord>` | no | One entry per tool call attempt. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `WeatherJob` (entity state) | `jobId` | `String` | no | — |
| | `query` | `Optional<WeatherQuery>` | yes | Populated after `JobQueued`. |
| | `report` | `Optional<WeatherReport>` | yes | Populated after `JobCompleted`. |
| | `status` | `JobStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `JobQueued` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `WeatherJob` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`InvocationMode`: `TRIGGER_AGENT`, `CALL_AGENT`.
`ToolCallStatus`: `EXECUTED`, `BLOCKED`, `FAILED`.
`JobStatus`: `QUEUED`, `RUNNING`, `COMPLETED`, `FAILED`.

## Events (`WeatherJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobQueued` | `query` | → QUEUED |
| `JobStarted` | — | → RUNNING |
| `JobCompleted` | `report` | → COMPLETED (terminal happy) |
| `JobFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `WeatherJob.initial("")` with all `Optional` fields as `Optional.empty()` and `status = QUEUED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`JobRow` mirrors `WeatherJob`. The view declares ONE query: `getAllJobs: SELECT * AS jobs FROM job_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`WeatherTasks.java`)

```java
public final class WeatherTasks {
  public static final Task<WeatherReport> FETCH_WEATHER_REPORT = Task
      .name("Fetch weather report")
      .description("Resolve the city, call the weather tools, and assemble a WeatherReport")
      .resultConformsTo(WeatherReport.class);

  private WeatherTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## GuardrailResult (supporting)

```java
record GuardrailResult(
    boolean pass,
    String rejectionCode,   // null when pass=true
    String rejectionDetail  // null when pass=true
) {}
```

Used internally by `ToolCallGuardrail.check(toolName, args)`. Not stored in the entity; the relevant detail flows into `ToolCallRecord.guardRejectionReason` when a call is blocked.
