# Data model — flight-booking-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `FlightOffer` | `flightId` | `String` | no | Stable id for this flight on this route/date. |
| | `carrier` | `String` | no | Airline name. |
| | `flightNumber` | `String` | no | IATA flight number (e.g. AA100). |
| | `origin` | `String` | no | IATA airport code. |
| | `destination` | `String` | no | IATA airport code. |
| | `departureAt` | `Instant` | no | Scheduled departure (UTC). |
| | `arrivalAt` | `Instant` | no | Scheduled arrival (UTC). |
| | `cabinClass` | `String` | no | ECONOMY / BUSINESS / FIRST. |
| | `fareUsdCents` | `int` | no | Base fare in US cents (no seat upgrade). |
| `FlightOfferSet` | `offers` | `List<FlightOffer>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `searchedAt` | `Instant` | no | When the SEARCH task returned. |
| `SeatMap` | `flightId` | `String` | no | Matches a `FlightOffer.flightId`. |
| | `seats` | `List<SeatInfo>` | no | All seats on the aircraft for this flight. |
| `SeatInfo` | `seatCode` | `String` | no | Row + letter (e.g. 22A). |
| | `cabinClass` | `String` | no | ECONOMY / BUSINESS / FIRST. |
| | `available` | `boolean` | no | False = already occupied or blocked. |
| | `upgradeCostUsdCents` | `int` | no | 0 for standard seats; positive for premium seats within the same cabin. |
| `RankedSeat` | `seatCode` | `String` | no | Winning seat code. |
| | `cabinClass` | `String` | no | Cabin class of the winning seat. |
| | `totalCostUsdCents` | `int` | no | `FlightOffer.fareUsdCents` + `SeatInfo.upgradeCostUsdCents`. |
| | `rationale` | `String` | no | One sentence explaining the ranking decision. |
| `SeatSelection` | `flightId` | `String` | no | Matches a `FlightOffer.flightId` from the upstream `FlightOfferSet`. |
| | `carrier` | `String` | no | Airline name. |
| | `flightNumber` | `String` | no | IATA flight number. |
| | `origin` | `String` | no | IATA airport code. |
| | `destination` | `String` | no | IATA airport code. |
| | `departureAt` | `Instant` | no | Scheduled departure (UTC). |
| | `arrivalAt` | `Instant` | no | Scheduled arrival (UTC). |
| | `seatCode` | `String` | no | Selected seat code. |
| | `cabinClass` | `String` | no | Selected cabin class. |
| | `totalCostUsdCents` | `int` | no | Total cost including any seat upgrade. |
| | `selectedAt` | `Instant` | no | When the SELECT task returned. |
| `ReservationId` | `reservationId` | `String` | no | Opaque id returned by `commitReservation`. |
| `BookingConfirmation` | `confirmationCode` | `String` | no | Airline confirmation code (e.g. AAXQ99). |
| | `passengerRef` | `String` | no | Passenger reference used for the reservation. |
| | `flightId` | `String` | no | Matches the `SeatSelection.flightId`. |
| | `carrier` | `String` | no | Airline name. |
| | `flightNumber` | `String` | no | IATA flight number. |
| | `origin` | `String` | no | IATA airport code. |
| | `destination` | `String` | no | IATA airport code. |
| | `departureAt` | `Instant` | no | Scheduled departure (UTC). |
| | `seatCode` | `String` | no | Confirmed seat code. |
| | `cabinClass` | `String` | no | Confirmed cabin class. |
| | `totalCostUsdCents` | `int` | no | Total committed cost. |
| | `committedAt` | `Instant` | no | When `commitReservation` completed. |
| `GuardrailRejection` | `phase` | `String` | no | `SEARCH` / `SELECT` / `BOOK`. |
| | `tool` | `String` | no | Name of the misordered or premature tool. |
| | `reason` | `String` | no | Structured reason from `BookingGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `BookingRecord` (entity state) | `bookingId` | `String` | no | — |
| | `origin` | `Optional<String>` | yes | Populated after `BookingCreated`. |
| | `destination` | `Optional<String>` | yes | Populated after `BookingCreated`. |
| | `travelDate` | `Optional<String>` | yes | Populated after `BookingCreated`. |
| | `flightOffers` | `Optional<FlightOfferSet>` | yes | Populated after `FlightsFound`. |
| | `seatSelection` | `Optional<SeatSelection>` | yes | Populated after `SeatSelected`. |
| | `confirmation` | `Optional<BookingConfirmation>` | yes | Populated after `BookingCommitted`. |
| | `status` | `BookingStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `BookingCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `BookingRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BookingStatus`: `CREATED`, `SEARCHING`, `SEARCH_DONE`, `SELECTING`, `SEAT_SELECTED`, `AWAITING_CONFIRMATION`, `CONFIRMED`, `BOOKING`, `COMMITTED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `BookingGuardrail`): `SEARCH`, `SELECT`, `BOOK`.

## Events (`BookingEntity`)

| Event | Payload | Transition |
|---|---|---|
| `BookingCreated` | `origin: String, destination: String, travelDate: String` | → CREATED |
| `SearchStarted` | — | → SEARCHING |
| `FlightsFound` | `flightOffers: FlightOfferSet` | → SEARCH_DONE |
| `SelectStarted` | — | → SELECTING |
| `SeatSelected` | `seatSelection: SeatSelection` | → SEAT_SELECTED |
| `AwaitingConfirmation` | — | → AWAITING_CONFIRMATION |
| `BookingConfirmed` | — | → CONFIRMED |
| `BookStarted` | — | → BOOKING |
| `BookingCommitted` | `confirmation: BookingConfirmation` | → COMMITTED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `BookingFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `BookingRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BookingRow` mirrors `BookingRecord` exactly. The UI fetches the full row via `GET /api/bookings/{id}` and streams updates via `GET /api/bookings/sse`.

The view declares ONE query: `getAllBookings: SELECT * AS bookings FROM booking_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`BookingTasks.java`)

```java
public final class BookingTasks {
  public static final Task<FlightOfferSet> SEARCH_FLIGHTS = Task
      .name("Search flights")
      .description("Find available flights on the given route and date by calling searchFlights and getFareDetails")
      .resultConformsTo(FlightOfferSet.class);

  public static final Task<SeatSelection> SELECT_SEAT = Task
      .name("Select seat")
      .description("Pick the best available seat on the chosen flight by calling getSeatMap and rankSeats")
      .resultConformsTo(SeatSelection.class);

  public static final Task<BookingConfirmation> COMMIT_BOOKING = Task
      .name("Commit booking")
      .description("Commit the selected seat by calling commitReservation and getConfirmationDetails")
      .resultConformsTo(BookingConfirmation.class);

  private BookingTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `SearchTools`, `SelectTools`, and `BookTools` carries a `Phase` constant. `BookingGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix, and additionally rejects `commitReservation` when `status ∉ {CONFIRMED, BOOKING}` (the HITL gate). The tool registry is built once at startup; the guardrail reads it for every call.
