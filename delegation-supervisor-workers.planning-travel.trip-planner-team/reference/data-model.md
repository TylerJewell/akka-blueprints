# Data model

Every record the generated system defines. Nullable lifecycle fields are `Optional<T>` (Lesson 6); Akka serializes `Optional<T>` as the raw value or `null`.

## TripPlan (TripEntity state + TripsView row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Trip id (workflow id). |
| `destination` | `String` | no | City or region. |
| `preferences` | `String` | no | Free-text traveler preferences. |
| `days` | `int` | no | Trip length in days. |
| `status` | `TripStatus` | no | Lifecycle state. |
| `createdAt` | `Instant` | no | Request time. |
| `localInsights` | `Optional<String>` | yes | Local-expert summary. |
| `insightsAt` | `Optional<Instant>` | yes | When insights were gathered. |
| `itinerary` | `Optional<String>` | yes | Day-by-day plan. |
| `itineraryAt` | `Optional<Instant>` | yes | When itinerary was drafted. |
| `logistics` | `Optional<String>` | yes | Transport/lodging/timing notes. |
| `logisticsAt` | `Optional<Instant>` | yes | When logistics were planned. |
| `finalPlan` | `Optional<String>` | yes | Assembled final plan. |
| `completedAt` | `Optional<Instant>` | yes | Completion time. |
| `failureReason` | `Optional<String>` | yes | Why the trip failed. |
| `failedAt` | `Optional<Instant>` | yes | Failure time. |

`emptyState()` returns `TripPlan.initial("", "", "", 0)` with placeholder identity — never references `commandContext()` (Lesson 3).

## TripStatus enum

`GATHERING_INSIGHTS | PLANNING_ITINERARY | PLANNING_LOGISTICS | COMPLETED | FAILED`

Note: `TripsView` has no `WHERE status` query — Akka cannot auto-index enum columns (Lesson 2). The single query `getAllTrips` returns all rows; callers filter client-side.

## Agent result records

| Record | Fields | Used by |
|---|---|---|
| `Insights` | `String summary`, `List<String> sources` | `LocalExpertAgent` |
| `Itinerary` | `int days`, `String text` | `ItineraryAgent` |
| `Logistics` | `String text` | `LogisticsAgent` |

## Events

| Event | Trigger |
|---|---|
| `TripRequested` | `TripEntity.request` when a workflow starts. |
| `InsightsGathered` | `recordInsights` after the gather step. |
| `ItineraryDrafted` | `recordItinerary` after the draft step. |
| `LogisticsPlanned` | `recordLogistics` after the plan step. |
| `TripCompleted` | `complete` after the finalize step assembles `finalPlan`. |
| `TripFailed` | `fail` when a step exhausts retries. |
| `TripRequestQueued` | `InboundRequestQueue.enqueue` from the simulator. |
