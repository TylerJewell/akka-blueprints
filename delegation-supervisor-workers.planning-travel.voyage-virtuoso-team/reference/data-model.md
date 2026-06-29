# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## TravelRequest (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `requestId` | `String` | no | UUID, also the workflow id |
| `destination` | `String` | no | Destination city and country |
| `origin` | `String` | no | Origin city and country |
| `tier` | `String` | no | Travel tier: economy / business / first |
| `status` | `RequestStatus` | no | Lifecycle state |
| `flightPlan` | `Optional<FlightPlan>` | yes | FlightSpecialist output; null until `FlightPlanAttached` |
| `lodgingPlan` | `Optional<LodgingPlan>` | yes | AccommodationSpecialist output; null until `LodgingPlanAttached` |
| `experiencePlan` | `Optional<ExperiencePlan>` | yes | ExperienceSpecialist output; null until `ExperiencePlanAttached` |
| `logisticsPlan` | `Optional<LogisticsPlan>` | yes | LogisticsSpecialist output; null until `LogisticsPlanAttached` |
| `itinerary` | `Optional<ItineraryPlan>` | yes | Assembled itinerary; null until `ItineraryAssembled` |
| `failureReason` | `Optional<String>` | yes | Set on `RequestBlocked` or `ItineraryPartial` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `EvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Request creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `TravelRequestView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TravelQuery(String destination, String origin, String departureDate,
                   String returnDate, int travellers, String tier, String requestedBy) {}

record AssemblyBrief(FlightPlan flight, LodgingPlan lodging,
                     ExperiencePlan experience, LogisticsPlan logistics) {}

record FlightPlan(String outboundRouting, String returnRouting, String fareClass,
                  String cabinClass, List<String> notes, Instant gatheredAt) {}

record LodgingPlan(String propertyName, String propertyCategory, String roomType,
                   List<String> highlights, Instant gatheredAt) {}

record ExperiencePlan(List<String> activities, List<String> dining,
                      List<String> cultural, Instant gatheredAt) {}

record LogisticsPlan(List<String> groundTransfers, List<String> entryRequirements,
                     List<String> travelAdvisories, Instant gatheredAt) {}

record ItineraryPlan(String synopsis, FlightPlan flightPlan, LodgingPlan lodgingPlan,
                     ExperiencePlan experiencePlan, LogisticsPlan logisticsPlan,
                     String guardrailVerdict, Instant assembledAt) {}
```

## Status enum

```java
enum RequestStatus { PLANNING, IN_PROGRESS, ASSEMBLED, PARTIAL, BLOCKED }
```

## Events

### TravelRequestEntity

| Event | Trigger |
|---|---|
| `RequestCreated` | Workflow creates the request (`createRequest`) |
| `FlightPlanAttached` | FlightSpecialist returns a `FlightPlan` |
| `LodgingPlanAttached` | AccommodationSpecialist returns a `LodgingPlan` |
| `ExperiencePlanAttached` | ExperienceSpecialist returns an `ExperiencePlan` |
| `LogisticsPlanAttached` | LogisticsSpecialist returns a `LogisticsPlan` |
| `ItineraryAssembled` | Director assembly passes the guardrail |
| `ItineraryPartial` | At least one specialist timed out; assembled from remaining outputs |
| `RequestBlocked` | Guardrail rejected the assembled itinerary |
| `EvalScored` | `EvalSampler` recorded a 1–5 score |

### RequestQueue

| Event | Trigger |
|---|---|
| `TravelSubmitted` | `enqueueTravel(destination, origin, tier, requestedBy)` from endpoint or simulator |

Fields: `{ requestId, destination, origin, tier, requestedBy, submittedAt }`.
