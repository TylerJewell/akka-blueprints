# Data model — restaurant-booking-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RestaurantInfo` | `restaurantId` | `String` | no | Stable catalogue identifier, e.g. `"bella-napoli"`. |
| | `name` | `String` | no | Display name. |
| | `cuisine` | `String` | no | Cuisine category, e.g. `"Italian"`. |
| | `address` | `String` | no | Street address. |
| | `operatingHours` | `TimeRange` | no | Open and close `LocalTime`. |
| | `maxPartySize` | `int` | no | Largest accepted party. Guardrail upper bound. |
| | `estimatedCoverCharge` | `int` | no | Per-person estimate in cents. |
| `TimeRange` | `open` | `LocalTime` | no | Opening time. |
| | `close` | `LocalTime` | no | Closing time. |
| `BookingRequest` | `bookingId` | `String` | no | UUID minted by `BookingEndpoint`. |
| | `rawRequest` | `String` | no | User's freeform text. Audit-only; not sent to the UI. |
| | `requestedBy` | `String` | no | User identifier. |
| | `initiatedAt` | `Instant` | no | When the endpoint received the request. |
| `BookingProposal` | `restaurantId` | `String` | no | MUST match a `RestaurantInfo.restaurantId`. |
| | `restaurantName` | `String` | no | As listed in the catalogue. |
| | `date` | `LocalDate` | no | Reservation date; MUST be in the future. |
| | `time` | `LocalTime` | no | Reservation time; MUST be within `operatingHours`. |
| | `partySize` | `int` | no | 1..`maxPartySize`. |
| | `contactName` | `String` | no | Full name from the request. |
| | `contactEmail` | `String` | no | Valid email address. |
| | `estimatedTotalCents` | `int` | no | `partySize` × `estimatedCoverCharge`. |
| | `proposedAt` | `Instant` | no | When the agent returned. |
| `BookingConfirmation` | `confirmationNumber` | `String` | no | `"CONF-"` + 6-hex stub-generated id. |
| | `confirmedAt` | `Instant` | no | When `BookingStub.createBooking` returned. |
| `Booking` (entity state) | `bookingId` | `String` | no | — |
| | `request` | `Optional<BookingRequest>` | yes | Populated after `BookingInitiated`. |
| | `proposal` | `Optional<BookingProposal>` | yes | Populated after `ConfirmationRequested`. |
| | `confirmation` | `Optional<BookingConfirmation>` | yes | Populated after `BookingCompleted`. |
| | `status` | `BookingStatus` | no | See enum. |
| | `failureReason` | `Optional<String>` | yes | Populated after `BookingFailed`. |
| | `createdAt` | `Instant` | no | When `BookingInitiated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Booking` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BookingStatus`: `INITIATED`, `COLLECTING`, `PENDING_CONFIRMATION`, `COMMITTING`, `CONFIRMED`, `DECLINED`, `FAILED`.

## Events (`BookingEntity`)

| Event | Payload | Transition |
|---|---|---|
| `BookingInitiated` | `request` | → INITIATED |
| `CollectionStarted` | — | → COLLECTING |
| `ConfirmationRequested` | `proposal` | → PENDING_CONFIRMATION |
| `BookingConfirmed` | — | → COMMITTING (user confirmed; waiting for stub) |
| `BookingDeclined` | — | → DECLINED (terminal) |
| `CommitStarted` | — | (internal — no status change, marks commit in flight) |
| `BookingCompleted` | `confirmation` | → CONFIRMED (terminal happy) |
| `BookingFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Booking.initial("")` with all `Optional` fields as `Optional.empty()` and `status = INITIATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BookingRow` mirrors `Booking` minus `request.rawRequest` (the audit log keeps that). The UI fetches the raw request on demand via `GET /api/bookings/{id}` and reads `request.rawRequest` from the JSON.

The view declares ONE query: `getAllBookings: SELECT * AS bookings FROM booking_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`BookingTasks.java`)

```java
public final class BookingTasks {
  public static final Task<BookingProposal> BOOK_RESTAURANT = Task
      .name("Book restaurant")
      .description("Extract reservation parameters from the user request and return a BookingProposal for the selected restaurant")
      .resultConformsTo(BookingProposal.class);

  private BookingTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
