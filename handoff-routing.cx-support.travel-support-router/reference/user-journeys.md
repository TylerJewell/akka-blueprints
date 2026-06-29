# User journeys — travel-support-router

## J1 — Flight request: end-to-end handoff to FlightSpecialist with passenger confirmation

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `RequestSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as flight-flavoured to drop.

**Expected:**
- The request appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject and any PII are not.
- Within ~10 s the routing block shows `category = FLIGHTS`, `confidence = high`, and a one-sentence reason. Status changes to `ROUTED` then `ROUTED_FLIGHTS`.
- Within ~5 s of routing, the guardrail clears (`status = GUARDRAIL_CLEARED`) and the confirmation block appears in the right column with a plain-language summary and a Confirm / Reject button pair.
- On clicking **Confirm**, the request transitions to `RESOLUTION_DRAFTED` then `RESOLVED`. The `FlightSpecialist` draft appears as the published response with a green "Published" stamp.
- Within ~10 s of the routing decision, the routing score chip shows a number 1–5 with a rationale.

## J2 — Hotel request: end-to-end handoff to HotelSpecialist

**Preconditions:** Same as J1. The seeded JSONL includes a hotel-flavoured request.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a hotel-flavoured request.

**Expected:**
- Routing emits `category = HOTELS`. Status `ROUTED_HOTELS`.
- `HotelSpecialist` returns a `Resolution` with action `BOOKING_CHANGED` or `INFO_PROVIDED`.
- Guardrail allows. Confirmation gate presents a summary. On **Confirm**, status `RESOLVED`.
- `FlightSpecialist`, `CarRentalSpecialist`, and `ExcursionSpecialist` are never invoked for this request.

## J3 — Ambiguous request: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous short request.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Routing emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `RequestEscalated`. Status `ESCALATED`.
- No specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the routing reason.
- A `RoutingScored` event still fires; the routing score chip appears (typically high, since defaulting to `UNCLEAR` was correct).

## J4 — Guardrail blocks a policy-violating tool call

**Preconditions:** The seeded JSONL includes one request engineered to produce a non-refundable cancellation outside the 24-hour change window (and the mock-responses file includes a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip request to drop and be routed as `FLIGHTS`.
2. Wait for `FlightSpecialist` to produce a draft proposing the cancellation.

**Expected:**
- Status reaches `ROUTED_FLIGHTS`.
- The guardrail block in the UI shows a red badge with violation `non-refundable-outside-window`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- No confirmation gate is presented — the block fires before the passenger sees the change summary.
- The right column shows the blocked draft and an Unblock button for operator override.

## J5 — Passenger rejects at the confirmation gate

**Preconditions:** A request has reached `AWAITING_CONFIRMATION` (guardrail cleared, confirmation summary visible).

**Steps:**
1. Click the **Reject** button in the right column.
2. The UI posts `POST /api/requests/{id}/confirm` with `{ "outcome": "REJECT" }`.

**Expected:**
- The request transitions immediately to `PASSENGER_REJECTED`. `finishedAt` is set.
- The right column replaces the Confirm / Reject buttons with a muted "Passenger rejected — no booking mutation applied." message.
- No booking mutation was applied at any point. The specialist's draft is preserved in `resolution` for audit but is not published.
- The guardrail verdict remains visible in the UI (audit record of what was checked and cleared).

## J6 — Routing score appears on every routed request

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new request through the routing step.

**Expected:**
- Within ~10 s of `RoutingDecided`, the per-request routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail view.
- A low score does not block the workflow — the guardrail and confirmation gate are the operative controls; the score is a continuous quality signal.
