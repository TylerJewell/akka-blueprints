# Data model — surprise-trip-planner

Every record the generated system defines. Nullable lifecycle fields are `Optional<T>` on the row record (Lesson 6).

## `Trip` (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Trip id (UUID) |
| `preferences` | `PreferenceSet` | no | The submitted preferences |
| `status` | `TripStatus` | no | Lifecycle state |
| `requestedAt` | `Optional<Instant>` | yes | When the request was recorded |
| `destinationName` | `Optional<String>` | yes | Chosen destination (withheld until READY) |
| `destinationRationale` | `Optional<String>` | yes | Why it matches the vibe |
| `researchedAt` | `Optional<Instant>` | yes | When the destination was chosen |
| `logistics` | `Optional<Logistics>` | yes | Transport + lodging plan |
| `activities` | `Optional<List<DayPlan>>` | yes | Day-by-day plan |
| `assembledAt` | `Optional<Instant>` | yes | When the itinerary was assembled |
| `groundingNotes` | `Optional<String>` | yes | G2 grounding adjustments |
| `blockedToolNote` | `Optional<String>` | yes | G1 blocked-tool record |
| `escalatedAt` | `Optional<Instant>` | yes | When the monitor escalated |
| `failureReason` | `Optional<String>` | yes | Why the workflow failed |

`Trip.initial(String id)` returns a blank trip in `REQUESTED` with all Optionals empty — no `commandContext()` reference (Lesson 3).

## Supporting records

| Record | Fields |
|---|---|
| `PreferenceSet` | `String vibe`, `String budgetBand`, `String startDate`, `String endDate`, `int partySize`, `List<String> noGo` |
| `Destination` | `String name`, `String rationale` |
| `Logistics` | `String transport`, `String lodging`, `String estCostBand` |
| `DayPlan` | `int day`, `String summary`, `List<String> items` |
| `Activities` | `List<DayPlan> days` |
| `Itinerary` | `String destinationName`, `String destinationRationale`, `Logistics logistics`, `List<DayPlan> activities`, `String groundingNotes` |

## `TripStatus` enum

`REQUESTED`, `RESEARCHED`, `READY`, `ESCALATED`, `FAILED`.

## Events (TripEntity)

| Event | Trigger |
|---|---|
| `TripRequested` | `recordRequest` — workflow starts |
| `DestinationChosen` | `recordDestination` — DestinationAgent result |
| `LogisticsPlanned` | `recordLogistics` — LogisticsAgent result |
| `ActivitiesPlanned` | `recordActivities` — ActivitiesAgent result |
| `ItineraryAssembled` | `recordItinerary` — supervisor assembly persisted |
| `ToolCallBlocked` | `recordToolBlock` — G1 blocked a worker tool call |
| `TripEscalated` | `markEscalated` — StalledTripMonitor |
| `TripFailed` | `markFailed` — workflow step exhausted retries |

## InboundRequestQueue

| Event | Trigger |
|---|---|
| `InboundRequestQueued` | `enqueueRequest(PreferenceSet)` from TripEndpoint or RequestSimulator |

## View row type

`TripsView` uses `Trip` as its row type. One query: `getAllTrips` → `SELECT * AS trips FROM trips_view`. No `WHERE status` filter (Lesson 2); callers filter client-side.
