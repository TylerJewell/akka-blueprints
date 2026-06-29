# User journeys — booking-support-agent

Five acceptance journeys. Each is numbered, has explicit preconditions, numbered steps, and an expected outcome. These define the bar that the generated system must clear.

---

## J1 — Customer looks up a flight departure time

**Preconditions:**
- Session created for customer `C001`.
- Booking `BK-001` exists in `BookingStore` with `customerId = C001`, `flightNumber = AA101`, `departureAt = 2026-07-03T07:15Z`.

**Steps:**
1. Customer types: "What time does my BK-001 flight depart?"
2. UI POSTs to `/api/sessions/{sessionId}/turns` with `rawMessage = "What time does my BK-001 flight depart?"`.
3. Turn appears in status `RECEIVED`.
4. Within ~1 s, turn transitions to `SANITIZED`; PII-category chips show none found.
5. Within ~30 s, turn transitions to `AGENT_REPLIED`.
6. Tool-call log shows one row: `lookUpBooking / BK-001 / ALLOWED`.
7. Reply text contains "07:15" and "AA101".
8. `updatedBooking` is null (no mutation occurred).

**Expected:** The agent answers with the departure time sourced from the booking record. No flight details beyond what `BookingStore` returned are present in the reply.

---

## J2 — Customer cancels a flexible-fare booking

**Preconditions:**
- Session created for customer `C001`.
- Booking `BK-003` exists with `customerId = C001`, `cancellationPermitted = true`, `bookingStatus = CONFIRMED`.

**Steps:**
1. Customer types: "Please cancel booking BK-003."
2. UI POSTs the message.
3. Turn progresses through `RECEIVED → SANITIZED → AGENT_REPLIED`.
4. Tool-call log shows two rows: `lookUpBooking / BK-003 / ALLOWED`, then `cancelBooking / BK-003 / ALLOWED`.
5. Reply text confirms the cancellation.
6. `updatedBooking` is present; its `bookingStatus` field is `CANCELLED`.

**Expected:** The booking is cancelled. The entity records an `AgentTurnCompleted` event with `updatedBooking.bookingStatus = CANCELLED`.

---

## J3 — Guardrail blocks cross-customer cancellation

**Preconditions:**
- Session created for customer `C001`.
- Booking `BK-007` exists with `customerId = C002`.

**Steps:**
1. Customer (C001) types: "Cancel booking BK-007."
2. UI POSTs the message.
3. Turn progresses through `RECEIVED → SANITIZED → AGENT_REPLIED`.
4. Tool-call log shows: `cancelBooking / BK-007 / BLOCKED / OWNERSHIP_VIOLATION`.
5. Reply text states the agent cannot process the request (does not mention that the booking belongs to another customer).
6. `updatedBooking` is null.

**Expected:** `cancelBooking` was never executed. The entity log contains no mutation event for `BK-007`. The customer receives a decline message without any cross-customer data disclosure.

---

## J4 — Guardrail blocks change on non-flexible fare

**Preconditions:**
- Session created for customer `C002`.
- Booking `BK-005` exists with `customerId = C002`, `seatChangePermitted = false`, `fareClass = W` (basic economy).

**Steps:**
1. Customer types: "I'd like to change my seat on BK-005 to an aisle seat."
2. UI POSTs the message.
3. Turn progresses through `RECEIVED → SANITIZED → AGENT_REPLIED`.
4. Tool-call log shows: `modifyBooking / BK-005 / BLOCKED / FARE_RULE_VIOLATION`.
5. Reply text explains that this fare does not allow seat changes and suggests contacting support for alternatives.

**Expected:** No seat change was applied. The entity log contains no modification event for `BK-005`.

---

## J5 — PII is redacted before the LLM call

**Preconditions:**
- Session created for customer `C003`.
- Booking `BK-008` exists with `customerId = C003`.

**Steps:**
1. Customer types: "My card number is 4111 1111 1111 1111. Can you look up BK-008?"
2. UI POSTs the message.
3. Turn transitions to `SANITIZED`. PII-category chips show `payment-card`.
4. Sanitized-message block shows `[REDACTED-PAYMENT-CARD]` in place of the card number.
5. Turn transitions to `AGENT_REPLIED`. Tool-call log shows `lookUpBooking / BK-008 / ALLOWED`.

**Expected:** The entity's `message.rawMessage` retains `4111 1111 1111 1111`. The `sanitized.sanitizedText` field and the LLM call log contain only `[REDACTED-PAYMENT-CARD]`. The agent's reply does not reference the card number.
