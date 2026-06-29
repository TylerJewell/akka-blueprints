# User journeys — flight-booking-pipeline

## J1 — Search, select, confirm, and commit a booking

**Preconditions:** Service running on declared port (`http://localhost:9149/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded route `SFO → JFK, 2026-09-15` has a matching `src/main/resources/sample-data/routes/sfo-jfk-20260915.json` file.

**Steps:**
1. Open `http://localhost:9149/` → App UI tab.
2. From the **Pick a seeded route** dropdown, pick `SFO → JFK · Sep 15 2026`.
3. Click **Find flights**.
4. When the card reaches `AWAITING_CONFIRMATION`, review the itinerary in the right pane.
5. Click **Confirm booking**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `SEARCHING` within 1 s more.
- Within ~20 s the card reaches `SEARCH_DONE`. The right pane's Flight offers table shows ≥ 2 rows; each row has a non-empty carrier, flight number, departure time, and fare.
- Within ~20 s more the card reaches `SEAT_SELECTED`, then `AWAITING_CONFIRMATION`. The right pane shows the Seat selection panel (flight number, seat code, cabin class, total cost) and the yellow **Confirm booking** button.
- After the user clicks **Confirm booking**, the card transitions to `CONFIRMED`, then `BOOKING`, then `COMMITTED` within ~20 s. The right pane shows the Booking confirmation panel with a non-empty `confirmationCode`.
- Total elapsed time (excluding user think time): ≤ 70 s.

## J2 — HITL gate blocks premature booking write

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `commit-booking.json` includes one entry whose `tool_calls` array starts with `commitReservation` called before `BookingConfirmed` is on record — this is the deliberately premature-commit entry.

**Steps:**
1. Submit any seeded route.
2. Wait for the card to reach `AWAITING_CONFIRMATION`. Do NOT click confirm.
3. Watch the network panel of the browser dev tools (`/api/bookings/sse`) for the `booking-rejection` event.

**Expected:**
- When `bookStep` starts (after the user eventually clicks confirm), the mock triggers the premature-commit entry on the first iteration. `BookingGuardrail` rejects the `commitReservation` call with reason `hitl-violation: booking not confirmed by user`; a `GuardrailRejected` event lands on the entity.
- The booking write NEVER reaches `BookTools.commitReservation` — there is no log line from the tool body for the rejected call.
- The agent's second iteration calls `commitReservation` again (this time `CONFIRMED` is on record) and succeeds. The card eventually reaches `COMMITTED`.
- The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected. The mock's `search-flights.json` includes one entry whose `tool_calls` array starts with `getSeatMap(...)` — a SELECT-phase tool called during the SEARCH phase.

**Steps:**
1. Submit any seeded route three times in a row.
2. Watch the third submission's lifecycle in the browser network panel.

**Expected:**
- On the third submission's `searchStep`, the agent's first iteration calls `getSeatMap`. `BookingGuardrail` rejects it with reason `phase-violation: getSeatMap requires status in {SEARCH_DONE, SELECTING} with flightOffers present, saw SEARCHING`; a `GuardrailRejected` event lands on the entity.
- The misordered call NEVER reaches `SelectTools.getSeatMap` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal search sequence (`searchFlights` + `getFareDetails`). The card eventually reaches `AWAITING_CONFIRMATION` as in J1.
- The card in the App UI shows a small red dot. The rejection-log strip shows the one rejected call.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded route.
2. Click confirm when the booking reaches `AWAITING_CONFIRMATION`.
3. Wait for `COMMITTED`.
4. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the bookingId.

**Expected:**
- The SEARCH task's log entries show only `searchFlights` and `getFareDetails` calls.
- The SELECT task's log entries show only `getSeatMap` and `rankSeats` calls.
- The BOOK task's log entries show only `commitReservation` and `getConfirmationDetails` calls.
- No cross-phase calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is SEARCH → SELECT → BOOK. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.
- Between SELECT and BOOK, a `workflow.step.start confirmStep` log line shows the workflow pausing; a `workflow.step.resume confirmStep` log line shows it resuming after the user confirmed.

## J5 — Idempotent confirmation

**Preconditions:** Service running. Any model provider. A booking in `AWAITING_CONFIRMATION` state.

**Steps:**
1. Click **Confirm booking** once.
2. Immediately click **Confirm booking** a second time (rapid double-click or re-POST via curl).

**Expected:**
- The second `POST /api/bookings/{id}/confirm` call returns `200 { bookingId, status: "CONFIRMED" }` (or `BOOKING` or `COMMITTED` if the workflow advanced quickly) — it does not return an error, and it does not trigger a second `bookStep`.
- The booking reaches `COMMITTED` with exactly one `BookingCommitted` event on the entity event log. There is no duplicate `BookingCommitted` event.
- The confirmation code is the same in both observations.

## J6 — No matching flights for a custom route

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/routes/<slug>.json` exists for the user's route.

**Steps:**
1. In the App UI, type `ZZZ` as origin, `QQQ` as destination, and any date.
2. Click **Find flights**.

**Expected:**
- `SearchTools.searchFlights` returns an empty list.
- The agent's SEARCH task returns a `FlightOfferSet` with `offers = []`.
- The workflow advances to `selectStep`. The SELECT task returns a `SeatSelection` with `flightId = "(no-flights)"` and cost fields zeroed (the agent's refusal behaviour per `prompts/booking-agent.md`).
- The workflow reaches `AWAITING_CONFIRMATION` with the zeroed seat selection visible. The user can choose not to confirm — the booking stays in `AWAITING_CONFIRMATION` until the timeout.
- Nothing crashes; the empty booking is honestly empty.
