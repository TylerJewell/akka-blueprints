# User journeys — restaurant-booking-agent

## J1 — Submit a request, confirm, and receive a confirmation number

**Preconditions:** Service running on declared port (`http://localhost:9279/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9279/` → App UI tab.
2. From the **Example requests** dropdown, pick `Bella Napoli — party of 4`.
3. Click **Load example** to fill the textarea.
4. Click **Start booking**.

**Expected:**
- A new card appears in the live list with status `INITIATED` within 1 s.
- The card transitions to `COLLECTING` within 1 s.
- Within 30 s the card reaches `PENDING_CONFIRMATION`. The right pane shows the proposal summary: restaurant name "Bella Napoli", a date in the future, time "19:00", party size 4, contact name and email from the request, and an estimated total in currency.
- **Confirm** and **Decline** buttons are visible in the right pane.
- The user clicks **Confirm**.
- The card transitions to `COMMITTING` within 1 s, then to `CONFIRMED` within 3 s.
- The right pane shows a green confirmation chip with a `CONF-` prefixed confirmation number and a confirmed-at timestamp.
- No confirm/decline buttons are visible once the card is `CONFIRMED`.

## J2 — Guardrail blocks an invalid proposal and agent retries

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `book-restaurant.json` includes a deliberately invalid entry with a past date.

**Steps:**
1. Submit exactly 3 bookings using the same example request (J1 steps × 3).
2. On the third submission, watch the lifecycle in the browser dev tools network panel (`/api/bookings/sse`).

**Expected:**
- The third booking's first agent iteration produces a proposal with a date in the past.
- The `before-tool-call` guardrail rejects it. The rejected proposal NEVER lands in `BookingEntity` — there is no `ConfirmationRequested` event with the invalid date.
- The agent loop retries on iteration 2 (or 3 if needed) and produces a valid proposal. The card transitions to `PENDING_CONFIRMATION` with a future date.
- The service log shows one `guardrail.reject` line per rejected iteration, naming the failing check (`date-in-past`).

## J3 — User declines; no external write occurs

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a booking request (J1 steps 1–4).
2. Wait for the card to reach `PENDING_CONFIRMATION`.
3. Click **Decline** instead of **Confirm**.

**Expected:**
- The card immediately transitions to `DECLINED`.
- The right pane shows "Booking declined by user" in a muted block.
- `BookingStub.createBooking` is never called. The stub call counter (visible in service debug logs) stays at zero for this booking.
- No `CommitStarted` or `BookingCompleted` event appears in the entity log for this bookingId.

## J4 — Stub conflict produces a FAILED booking

**Preconditions:** Service running. The `BookingStub` is configured to return a conflict error on a pseudo-random 20% of bookings (keyed by bookingId).

**Steps:**
1. Submit several booking requests until one triggers the stub's conflict path (or directly submit the seeded bookingId whose stub deterministically returns an error — see `src/main/resources/sample-events/booking-requests.jsonl`).
2. Confirm the proposal when it appears.

**Expected:**
- After the user confirms, the card transitions to `COMMITTING`.
- The booking stub returns a conflict error.
- The card transitions to `FAILED`. The right pane shows the failure reason: "Stub returned conflict: venue not available for this time slot."
- No confirmation number is issued.
- The entity's `failureReason` field is populated; `finishedAt` is set.

## J5 — Confirmation timeout produces a FAILED booking

**Preconditions:** Service running with `confirmStep.stepTimeout` reduced to 10 s for testing (adjust in `application.conf` before running).

**Steps:**
1. Submit a booking request and wait for `PENDING_CONFIRMATION`.
2. Do not click **Confirm** or **Decline**.
3. Wait 15 s.

**Expected:**
- After the step timeout expires, `BookingWorkflow` transitions to its error step.
- The entity transitions to `FAILED` with `failureReason = "confirmation-timeout"`.
- The card updates to `FAILED` via SSE.
- No external write occurred.
