# Data model — maps-trip-planner

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TripRequest` | `tripId` | `String` | no | UUID minted by `PlanningEndpoint`. |
| | `originCity` | `String` | no | User-supplied departure city. |
| | `destinations` | `List<String>` | no | One or more destination city/region names. |
| | `startDate` | `LocalDate` | no | First travel day (ISO-8601). |
| | `endDate` | `LocalDate` | no | Last travel day, inclusive. |
| | `partySize` | `int` | no | Number of travellers. |
| | `preferencesText` | `String` | no | Free-text preferences (may be empty string). |
| | `requestedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ScreenResult` | `decision` | `ScreenDecision` | no | Enum: PASS or BLOCKED. |
| | `reason` | `String` | no | "ok" on PASS; block reason on BLOCKED. |
| `PlaceStop` | `placeName` | `String` | no | Resolved place name. |
| | `geocodedAddress` | `String` | no | Full address from geocode tool. |
| | `latDeg` | `double` | no | Latitude in decimal degrees (0.0 if geocoding failed). |
| | `lngDeg` | `double` | no | Longitude in decimal degrees (0.0 if geocoding failed). |
| | `visitMinutes` | `int` | no | Estimated visit duration; must be > 0. |
| | `description` | `String` | no | Short prose description. |
| `TransitLeg` | `fromPlace` | `String` | no | Must match preceding stop's `placeName`. |
| | `toPlace` | `String` | no | Must match next stop's `placeName`. |
| | `mode` | `TransitMode` | no | Enum value. |
| | `durationMinutes` | `int` | no | Must be > 0 (guardrail rejects ≤ 0). |
| `ItineraryDay` | `dayNumber` | `int` | no | 1-indexed. |
| | `date` | `LocalDate` | no | Calendar date for this day. |
| | `stops` | `List<PlaceStop>` | no | At least 2 per day (guardrail enforced). |
| | `legs` | `List<TransitLeg>` | no | One leg between each consecutive pair of stops. |
| | `dayNarrative` | `String` | no | 2–3 sentence description of the day's theme. |
| `TripItinerary` | `quality` | `ItineraryQuality` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `days` | `List<ItineraryDay>` | no | One entry per calendar day. |
| | `preferencesAddressed` | `List<String>` | no | Preference keywords from request that appear in the itinerary. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `CoverageEval` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `CoverageScorer` finished. |
| `TripPlan` (entity state) | `tripId` | `String` | no | — |
| | `request` | `Optional<TripRequest>` | yes | Populated after `TripRequestSubmitted`. |
| | `screen` | `Optional<ScreenResult>` | yes | Populated after `RequestScreened` or `RequestBlocked`. |
| | `itinerary` | `Optional<TripItinerary>` | yes | Populated after `ItineraryRecorded`. |
| | `eval` | `Optional<CoverageEval>` | yes | Populated after `EvaluationScored`. |
| | `status` | `TripStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TripRequestSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `TripPlan` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ScreenDecision`: `PASS`, `BLOCKED`.
`TransitMode`: `WALK`, `DRIVE`, `TRANSIT`, `CYCLE`.
`ItineraryQuality`: `EXCELLENT`, `GOOD`, `PARTIAL`.
`TripStatus`: `SUBMITTED`, `SCREENED`, `BLOCKED`, `PLANNING`, `ITINERARY_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`TripRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TripRequestSubmitted` | `request` | → SUBMITTED |
| `RequestScreened` | `screen` (PASS) | → SCREENED |
| `RequestBlocked` | `reason: String` | → BLOCKED (terminal) |
| `PlanningStarted` | — | → PLANNING |
| `ItineraryRecorded` | `itinerary` | → ITINERARY_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `PlanningFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `TripPlan.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`TripRow` mirrors `TripPlan` minus the raw `preferencesText` (the view stores a trimmed keyword summary derived from `preferencesAddressed` once the itinerary lands; the full text is available on demand via `GET /api/trips/{id}`).

The view declares ONE query: `getAllTrips: SELECT * AS trips FROM itinerary_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`TripTasks.java`)

```java
public final class TripTasks {
  public static final Task<TripItinerary> PLAN_TRIP = Task
      .name("Plan trip")
      .description("Use Maps tools to geocode destinations and produce a TripItinerary with one ItineraryDay per requested travel date")
      .resultConformsTo(TripItinerary.class);

  private TripTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
