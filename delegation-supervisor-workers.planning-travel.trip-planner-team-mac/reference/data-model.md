# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## Trip (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `tripId` | `String` | no | UUID, also the workflow id |
| `request` | `TripRequest` | no | The submitted trip parameters |
| `status` | `TripStatus` | no | Lifecycle state |
| `destinationNotes` | `Optional<DestinationNotes>` | yes | Researcher output; null until `DestinationResearched` |
| `itinerary` | `Optional<Itinerary>` | yes | Planner output; null until `ItineraryDrafted` |
| `bookingProposals` | `Optional<List<BookingProposal>>` | yes | Booking output; null until `BookingsPrepared` |
| `plan` | `Optional<TripPlan>` | yes | Compiled plan; null until `PlanCompiled` |
| `failureReason` | `Optional<String>` | yes | Set on `TripBlocked` or `TripDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `PlanEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Trip creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition (CONFIRMED, REJECTED, DEGRADED, BLOCKED) |

The `TripView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TripRequest(
    String destination,
    String startDate,
    String endDate,
    int travellerCount,
    double budgetUsd,
    String requestedBy
) {}

record TripDecomposition(
    String researchQuery,
    String itineraryBrief,
    String bookingScope
) {}

record DestinationNotes(
    String climate,
    String visaRequirements,
    List<String> highlights,
    List<String> localTips,
    Instant researchedAt
) {}

record ItineraryDay(
    int day,
    String title,
    String description,
    List<String> activities
) {}

record Itinerary(
    List<ItineraryDay> days,
    String overallTheme,
    Instant draftedAt
) {}

record BookingProposal(
    String type,
    String description,
    double estimatedCostUsd,
    String referenceCode,
    Instant preparedAt
) {}

record TripPlan(
    Itinerary itinerary,
    DestinationNotes destinationNotes,
    List<BookingProposal> bookingProposals,
    String compilationNotes,
    Instant compiledAt
) {}
```

## Status enum

```java
enum TripStatus {
    PLANNING, IN_PROGRESS, AWAITING_APPROVAL, CONFIRMED, REJECTED, DEGRADED, BLOCKED
}
```

## Events

### TripEntity

| Event | Trigger |
|---|---|
| `TripCreated` | Workflow creates the trip (`createTrip`) |
| `DestinationResearched` | DestinationResearcher returns `DestinationNotes` |
| `ItineraryDrafted` | ItineraryPlanner returns `Itinerary` |
| `BookingsPrepared` | BookingAgent returns `List<BookingProposal>` |
| `PlanCompiled` | TripSupervisor returns compiled `TripPlan` |
| `ApprovalRequested` | Workflow issues approval token; trip moves to AWAITING_APPROVAL |
| `TripConfirmed` | User approves via `POST /api/trips/{id}/approve` |
| `TripRejected` | User rejects via `POST /api/trips/{id}/reject` |
| `TripDegraded` | A worker timed out; compiled from partial inputs |
| `TripBlocked` | Budget guardrail rejected the plan |
| `PlanEvalScored` | `EvalSampler` recorded a 1–5 score |

### ApprovalEntity

| Event | Trigger |
|---|---|
| `ApprovalTokenIssued` | `issueToken(tripId)` from the workflow's approval step |
| `ApprovalDecisionRecorded` | `recordDecision(tripId, decision, reason)` from the endpoint |

Fields on `ApprovalTokenIssued`: `{ tripId, token, issuedAt }`.
Fields on `ApprovalDecisionRecorded`: `{ tripId, decision, decidedAt }`.

### TripRequestQueue

| Event | Trigger |
|---|---|
| `TripRequestSubmitted` | `enqueueRequest(TripRequest)` from endpoint or simulator |

Fields: `{ tripId, destination, startDate, endDate, travellerCount, budgetUsd, requestedBy, submittedAt }`.
