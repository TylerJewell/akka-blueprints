# Data model — weather-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WeatherQuestion` | `questionId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `questionText` | `String` | no | User's natural-language question. |
| | `units` | `UnitSystem` | no | Requested unit system for temperatures and wind. |
| | `submittedBy` | `String` | no | User identifier (optional text field; defaults to `"anonymous"`). |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `GeocodingResult` | `displayName` | `String` | no | Human-readable location name returned by the geocoder. |
| | `latitude` | `double` | no | Decimal degrees, WGS 84. |
| | `longitude` | `double` | no | Decimal degrees, WGS 84. |
| `CurrentConditions` | `temperatureDegrees` | `double` | no | In the unit system requested. |
| | `feelsLikeDegrees` | `double` | no | Apparent temperature. |
| | `windSpeedKmh` | `double` | no | Always km/h internally; formatted per unit in the UI. |
| | `humidityPercent` | `int` | no | 0–100. |
| | `description` | `String` | no | Short free-text description, e.g., `"partly cloudy"`. |
| | `iconCode` | `String` | no | Icon identifier for UI rendering, e.g., `"02d"`. |
| `ForecastDay` | `date` | `String` | no | ISO-8601 date string, e.g., `"2026-07-01"`. |
| | `highDegrees` | `double` | no | Daily high in requested unit. |
| | `lowDegrees` | `double` | no | Daily low in requested unit. |
| | `precipitationMm` | `double` | no | Expected precipitation. |
| | `description` | `String` | no | Short description. |
| `WeatherAnswer` | `questionId` | `String` | no | Matches the originating `WeatherQuestion`. |
| | `locationDisplay` | `String` | no | From geocoding result's `displayName`. |
| | `latitude` | `double` | no | From geocoding result. |
| | `longitude` | `double` | no | From geocoding result. |
| | `units` | `UnitSystem` | no | Unit system used. |
| | `current` | `Optional<CurrentConditions>` | yes | Populated when a current-conditions call was made. |
| | `forecast` | `Optional<List<ForecastDay>>` | yes | Populated when a multi-day forecast was requested. |
| | `narrative` | `String` | no | 1–3 sentence natural-language summary. |
| | `answeredAt` | `Instant` | no | When `WeatherAgent` returned. |
| `ToolCallRecord` | `toolName` | `String` | no | `"geocode"` or `"getWeather"`. |
| | `parameters` | `Map<String, String>` | no | Parameter names and string-serialised values. |
| | `outcome` | `String` | no | One of `"accepted"`, `"rejected"`, `"success"`, `"error"`. |
| | `calledAt` | `Instant` | no | When the tool call was attempted. |
| `Query` (entity state) | `questionId` | `String` | no | — |
| | `question` | `Optional<WeatherQuestion>` | yes | Populated after `QuestionSubmitted`. |
| | `geocoding` | `Optional<GeocodingResult>` | yes | Populated after the agent's first successful geocode call (via `ToolCallRecorded`). |
| | `answer` | `Optional<WeatherAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `toolCalls` | `List<ToolCallRecord>` | no | Grows with each `ToolCallRecorded` event. Empty list initially. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuestionSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `failureReason` | `Optional<String>` | yes | Populated on `QueryFailed`. |

Every nullable field on `Query` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`UnitSystem`: `METRIC`, `IMPERIAL`, `STANDARD`.
`QueryStatus`: `SUBMITTED`, `PROCESSING`, `ANSWERED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuestionSubmitted` | `question` | → SUBMITTED |
| `ProcessingStarted` | — | → PROCESSING |
| `ToolCallRecorded` | `toolCall` | stays in PROCESSING (appends to `toolCalls` list) |
| `AnswerRecorded` | `answer` | → ANSWERED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Query.initial("")` with all `Optional` fields as `Optional.empty()`, `toolCalls = List.of()`, and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `Query` in full. No fields are stripped (unlike DocReview's omission of `rawDocument`; weather queries contain no sensitive content). The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE` filter (Lesson 2); the endpoint sorts newest-first client-side.

## Task definition (`WeatherTasks.java`)

```java
public final class WeatherTasks {
  public static final Task<WeatherAnswer> ANSWER_WEATHER_QUESTION = Task
      .name("Answer weather question")
      .description("Resolve the location and fetch current conditions or a forecast, then return a WeatherAnswer")
      .resultConformsTo(WeatherAnswer.class);

  private WeatherTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
