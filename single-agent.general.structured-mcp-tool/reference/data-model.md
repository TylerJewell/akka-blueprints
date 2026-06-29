# Data model — structured-mcp-tool

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `location` | `String` | no | User-supplied location name. |
| | `unit` | `UnitPreference` | no | CELSIUS or FAHRENHEIT. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `WeatherPayload` | `location` | `String` | no | Echo of the requested location from the MCP tool. |
| | `temperatureCelsius` | `double` | no | Current temperature in °C. |
| | `humidityPercent` | `int` | no | Relative humidity 0–100. |
| | `windSpeedKph` | `double` | no | Wind speed in km/h. |
| | `condition` | `WeatherCondition` | no | Enum value. |
| | `observedAt` | `String` | no | ISO-8601 observation timestamp from the MCP tool. |
| `WeatherSummary` | `location` | `String` | no | — |
| | `temperatureCelsius` | `double` | no | — |
| | `temperatureFahrenheit` | `Optional<Double>` | yes | Populated only when unit = FAHRENHEIT. |
| | `humidityPercent` | `int` | no | — |
| | `windSpeedKph` | `double` | no | — |
| | `condition` | `WeatherCondition` | no | Enum value. |
| | `narrative` | `String` | no | 1-sentence agent-generated description. |
| | `returnedAt` | `Instant` | no | When `WeatherQueryAgent` returned the summary. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `summary` | `Optional<WeatherSummary>` | yes | Populated after `SummaryRecorded`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`WeatherCondition`: `CLEAR`, `PARTLY_CLOUDY`, `OVERCAST`, `RAIN`, `THUNDERSTORM`, `SNOW`, `FOG`.
`UnitPreference`: `CELSIUS`, `FAHRENHEIT`.
`QueryStatus`: `SUBMITTED`, `RUNNING`, `COMPLETED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `QueryStarted` | — | → RUNNING |
| `SummaryRecorded` | `summary` | → COMPLETED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query`. The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`WeatherQueryTasks.java`)

```java
public final class WeatherQueryTasks {
  public static final Task<WeatherSummary> GET_WEATHER_SUMMARY = Task
      .name("Get weather summary")
      .description("Call get_weather_info for the requested location and return a structured WeatherSummary")
      .resultConformsTo(WeatherSummary.class);

  private WeatherQueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## MCP tool schema (`InlineMcpServer`)

The `get_weather_info` tool is registered with the following input schema:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `location` | `string` | yes | City or region name. |
| `unit` | `string` | yes | `"CELSIUS"` or `"FAHRENHEIT"`. |

Return type: `WeatherPayload` (see record table above). The MCP server returns the payload as JSON; the `after-tool-call` guardrail validates it before it reaches the agent's reasoning step.
