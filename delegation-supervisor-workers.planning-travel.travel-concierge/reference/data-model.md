# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## Trip (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `tripId` | `String` | no | UUID, also the workflow id |
| `destination` | `String` | no | The requested destination |
| `departureDateIso` | `String` | no | ISO-8601 date (YYYY-MM-DD) |
| `returnDateIso` | `String` | no | ISO-8601 date (YYYY-MM-DD) |
| `travellerCount` | `int` | no | Number of travellers |
| `budgetUsd` | `int` | no | User's stated maximum spend in USD |
| `status` | `TripStatus` | no | Lifecycle state |
| `destinationBrief` | `Optional<DestinationBrief>` | yes | Scout output; null until `DestinationBriefAttached` |
| `fareEstimate` | `Optional<FareEstimate>` | yes | FareAgent output; null until `FareEstimateAttached` |
| `itinerary` | `Optional<TripItinerary>` | yes | Assembled itinerary; null until `ItineraryAssembled` |
| `bookingReference` | `Optional<String>` | yes | External reference; null until `TripConfirmed` |
| `failureReason` | `Optional<String>` | yes | Set on `TripBlocked` or `TripDegraded` |
| `createdAt` | `Instant` | no | Trip creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `TripView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TripRequest(String destination, String departureDateIso, String returnDateIso,
                   int travellerCount, int budgetUsd, String requestedBy) {}

record TripPlan(String destinationQuery, String fareQuery) {}

record DestinationBrief(String destination, List<String> highlights,
                        List<String> advisories, String visaRequirement,
                        Instant researchedAt) {}

record FareEstimate(int flightFareUsd, int accommodationFareUsd,
                    int totalFareUsd, String currency, Instant estimatedAt) {}

record TripItinerary(String destination, String departureDateIso, String returnDateIso,
                     List<String> highlights, List<String> advisories,
                     String visaRequirement, FareEstimate fare,
                     Optional<String> warnings, Instant assembledAt) {}
```

## Status enum

```java
enum TripStatus { PLANNING, RESEARCHING, AWAITING_APPROVAL, CONFIRMED, DECLINED, DEGRADED, BLOCKED }
```

## Events

### TripEntity

| Event | Trigger |
|---|---|
| `TripCreated` | Workflow creates the trip (`createTrip`) |
| `ResearchStarted` | Workflow begins parallel worker phase |
| `DestinationBriefAttached` | DestinationScout returns a `DestinationBrief` |
| `FareEstimateAttached` | FareAgent returns a `FareEstimate` |
| `ItineraryAssembled` | Coordinator assembles the final `TripItinerary` |
| `ApprovalRequested` | Workflow enters HITL pause (`AWAITING_APPROVAL`) |
| `TripConfirmed` | User confirmed + guardrail passed + booking committed |
| `TripDeclined` | User declined at the HITL step |
| `TripDegraded` | A worker timed out; assembled from partial input |
| `TripBlocked` | Guardrail rejected the booking payload |

### TripRequestQueue

| Event | Trigger |
|---|---|
| `TripRequested` | `enqueueTripRequest(...)` from endpoint or simulator |

Fields: `{ tripId, destination, departureDateIso, returnDateIso, travellerCount, budgetUsd, requestedBy, submittedAt }`.
