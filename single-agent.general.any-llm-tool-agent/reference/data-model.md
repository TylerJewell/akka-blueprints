# Data model — any-llm-tool-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WeatherQuery` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `queryText` | `String` | no | User's raw natural-language query. |
| | `requestedBackend` | `String` | no | Backend name chosen in the UI (e.g., `mock`, `anthropic`). Stored for traceability only. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `WeatherData` | `location` | `String` | no | Location string passed to `get_weather`. |
| | `condition` | `Condition` | no | Enum value. |
| | `temperatureCelsius` | `double` | no | Current temperature in Celsius. |
| | `humidityPercent` | `int` | no | Relative humidity 0–100. |
| | `windSpeedKmh` | `double` | no | Wind speed in km/h. |
| `WeatherReport` | `location` | `String` | no | Resolved location (matches `WeatherData.location`). |
| | `data` | `WeatherData` | no | Raw data from `WeatherTools.get_weather`. |
| | `narrative` | `String` | no | One-sentence summary. |
| | `reportedAt` | `Instant` | no | When the agent returned. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `query` | `Optional<WeatherQuery>` | yes | Populated after `QuerySubmitted`. |
| | `report` | `Optional<WeatherReport>` | yes | Populated after `ReportRecorded`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Condition`: `SUNNY`, `CLOUDY`, `RAINY`, `SNOWY`, `UNKNOWN`.
`QueryStatus`: `SUBMITTED`, `INVOKING`, `RESULT_RECORDED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `query` | → SUBMITTED |
| `InvocationStarted` | — | → INVOKING |
| `ReportRecorded` | `report` | → RESULT_RECORDED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query`. No field is excluded — the query text is not sensitive in this domain. The UI fetches a specific row via `GET /api/queries/{id}` when the user selects a card.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<WeatherReport> WEATHER_REPORT = Task
      .name("Weather report")
      .description("Look up the weather for the location in the user query and return a WeatherReport")
      .resultConformsTo(WeatherReport.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Tool class (`WeatherTools.java`)

```java
public class WeatherTools {
  @Tool("get_weather")
  public WeatherData get_weather(String location) {
    // Stub implementation — replace with a live HTTP call for production use.
    Condition[] conditions = { Condition.SUNNY, Condition.CLOUDY, Condition.RAINY };
    Condition condition = conditions[Math.abs(location.hashCode()) % conditions.length];
    double temp = 15 + (location.length() % 20);
    return new WeatherData(location, condition, temp, 60, 10.0);
  }
}
```

`WeatherTools` is NOT an Akka component. It is a plain Java class annotated so the agent framework discovers its `@Tool`-annotated methods. The `WeatherToolGuardrail` hook runs before this method body executes.
