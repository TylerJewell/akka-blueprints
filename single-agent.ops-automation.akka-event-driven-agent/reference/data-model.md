# Data model — ambient-durable-agent-pubsub

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WeatherRequestPayload` | `requestId` | `String` | no | UUID minted by `RequestEndpoint`. |
| | `location` | `String` | no | Location name (may be redacted in `SanitizedPayload`). |
| | `requesterContact` | `String` | yes | Email or identifier of the requesting party; may contain PII. |
| | `horizon` | `ForecastHorizon` | no | Forecast time window. |
| | `alertCategories` | `List<String>` | no | Categories to evaluate for hazard flag (empty list is valid). |
| | `receivedAt` | `Instant` | no | When the topic message arrived at `TopicConsumer`. |
| `ValidationResult` | `valid` | `boolean` | no | `true` = payload passed all guardrail checks. |
| | `violationCodes` | `List<String>` | no | Empty when `valid = true`; named codes on failure. |
| `SanitizedPayload` | `location` | `String` | no | Location with PII substrings redacted. |
| | `horizon` | `ForecastHorizon` | no | Unchanged from raw payload. |
| | `alertCategories` | `List<String>` | no | Unchanged from raw payload. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","name","host"]`. |
| `TemperatureRange` | `lowCelsius` | `int` | no | Lowest temperature expected in the horizon. |
| | `highCelsius` | `int` | no | Highest temperature expected in the horizon. |
| `WeatherReport` | `conditionsSummary` | `String` | no | 1–2 sentences describing expected conditions. |
| | `temperatureRange` | `TemperatureRange` | no | Low/high for the horizon window. |
| | `windSpeedKph` | `int` | no | Representative wind speed in kph. |
| | `precipitationPct` | `int` | no | 0–100 probability of precipitation. |
| | `hazardFlag` | `boolean` | no | `true` when at least one `alertCategory` is triggered. |
| | `hazardDetail` | `String` | no | Empty string when `hazardFlag = false`. |
| | `forecastedAt` | `Instant` | no | When `WeatherAgent` returned. |
| `Request` (entity state) | `requestId` | `String` | no | — |
| | `rawPayload` | `Optional<WeatherRequestPayload>` | yes | Populated after `MessageReceived`. |
| | `validation` | `Optional<ValidationResult>` | yes | Populated after `PayloadValidated`. |
| | `sanitized` | `Optional<SanitizedPayload>` | yes | Populated after `PayloadSanitized`. |
| | `report` | `Optional<WeatherReport>` | yes | Populated after `ReportRecorded`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Request` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ForecastHorizon`: `HOURLY_6H`, `DAILY_3D`, `WEEKLY_7D`.
`RequestStatus`: `RECEIVED`, `VALIDATED`, `SANITIZED`, `FORECASTING`, `COMPLETED`, `FAILED`.

## Events (`RequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MessageReceived` | `payload: WeatherRequestPayload` | → RECEIVED |
| `PayloadValidated` | `validation: ValidationResult` | → VALIDATED (valid=true) or → FAILED (valid=false) |
| `PayloadSanitized` | `sanitized: SanitizedPayload` | → SANITIZED |
| `ForecastingStarted` | — | → FORECASTING |
| `ReportRecorded` | `report: WeatherReport` | → COMPLETED (terminal happy) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Request.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RequestRow` mirrors `Request` minus `rawPayload.requesterContact` — the audit log retains the raw contact field. The UI fetches the full audit record on demand via `GET /api/requests/{id}` and reads `rawPayload.requesterContact` from the JSON.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM request_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ForecastTasks.java`)

```java
public final class ForecastTasks {
  public static final Task<WeatherReport> FORECAST_WEATHER = Task
      .name("Forecast weather")
      .description("Analyse the sanitized request payload and produce a WeatherReport")
      .resultConformsTo(WeatherReport.class);

  private ForecastTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Violation codes (`PayloadGuardrail`)

| Code | Check |
|---|---|
| `location.required` | `location` is null or blank |
| `location.too-long` | `location.length() > 256` |
| `horizon.invalid` | `horizon` is not a valid `ForecastHorizon` enum value |
| `alertCategories.null` | `alertCategories` is null |
| `requesterContact.injection` | `requesterContact` contains `< > ; -- ' "` |
