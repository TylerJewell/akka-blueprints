# User journeys — restaurant-assistant

Acceptance bar for the generated system. Each journey maps to a numbered acceptance item in SPEC.md Section 10.

---

## J1 — Menu query (no write)

**Preconditions:**
- Service is running.
- `menu-items.jsonl` seed is loaded (12 items including at least 2 vegetarian mains).

**Steps:**
1. Open the App UI tab and click **New session**.
2. Type: `What vegetarian main courses do you have?`
3. Click **Send**.
4. Wait for the ASSISTANT bubble to appear.

**Expected:**
- The agent lists only the vegetarian main courses from the seeded catalog (wild mushroom risotto, roasted beetroot tart) — no other items, no invented dishes.
- Prices quoted match the catalog exactly.
- Session status is `ACTIVE`.
- No tool call fires; no reservation or order write lands on the entity.
- The `ResponseGuardrail` validation passes on the first iteration (no retry visible in logs).

---

## J2 — Reservation (happy path)

**Preconditions:**
- Service is running.
- A session is open (OPEN or ACTIVE state).

**Steps:**
1. Type: `I'd like to book a table for 2 on Friday 4 July at 7 pm. My name is Alex Patel, phone 07700900123.`
2. Click **Send**.
3. Agent replies with a confirmation summary. Type: `Yes, please confirm.`
4. Click **Send**.

**Expected:**
- Agent calls `makeReservation` with `date="2026-07-04"`, `time="19:00"`, `partySize=2`, `guestName="Alex Patel"`, `contactPhone="07700900123"`.
- `ToolCallGuardrail` approves (date in future, party size in [1, 20]).
- `SessionEntity` emits `ReservationConfirmed`.
- Session status transitions to `RESERVATION_HELD`.
- An inline reservation confirmation card appears in the chat thread.
- SSE event with status `RESERVATION_HELD` received by the live list.

---

## J3 — Reservation with invalid party size (guardrail rejection)

**Preconditions:**
- Service is running.
- A session is open.

**Steps:**
1. Type: `Book a table for 30 people next Saturday at 8 pm. Name is Jordan Lee, 07911123456.`
2. Click **Send**.

**Expected:**
- Agent attempts to call `makeReservation` with `partySize=30`.
- `ToolCallGuardrail` rejects: party size 30 exceeds the maximum of 20.
- The agent receives the rejection reason and retries with a corrected reply — not a corrected write, but an apology to the customer asking for a valid party size.
- The customer sees: `I'm sorry, we can only accept reservations for up to 20 guests. Could you let me know the party size?`
- No `ReservationConfirmed` event lands on `SessionEntity`.
- Session status remains `ACTIVE`.

---

## J4 — Agent cites a non-existent menu item (response guardrail rejection + retry)

**Preconditions:**
- Service is running with mock LLM enabled (option a).
- The session id (modulo seed) falls on the 4th session so the mock selects the invalid response entry.

**Steps:**
1. Open the App UI tab and open session number 4 (or use the "Load seeded example → guardrail demo" option).
2. Type: `What's on the menu tonight?`
3. Click **Send**.

**Expected:**
- On the first iteration, `MockModelProvider` returns an `AssistantResponse` whose `replyText` cites a menu item not present in the catalog (e.g., "truffle gnocchi").
- `ResponseGuardrail` rejects the response; the rejection is logged but not shown to the customer.
- On the second iteration, the agent returns an accurate reply citing only real catalog items.
- The customer sees only the second, valid reply.
- Application logs show one `GUARDRAIL_REJECTED` event followed by `GUARDRAIL_ACCEPTED`.

---

## J5 — Order placement (happy path)

**Preconditions:**
- Service is running.
- A session is in ACTIVE or RESERVATION_HELD state.

**Steps:**
1. Type: `I'd like to order one wild mushroom risotto and two house whites.`
2. Click **Send**.
3. Agent replies with an order summary and total. Type: `That's right, place the order.`
4. Click **Send**.

**Expected:**
- Agent calls `submitOrder` with two `OrderLineItem` entries: one for the risotto (quantity 1) and one for house white wine (quantity 2), both using valid catalog `itemId` values.
- `ToolCallGuardrail` approves all line items.
- `SessionEntity` emits `OrderCommitted`.
- Session status transitions to `ORDER_PLACED`.
- An inline order summary card appears in the chat thread showing line items and total in GBP.
- SSE event with status `ORDER_PLACED` received by the live list.

---

## J6 — Session close

**Preconditions:**
- Service is running.
- A session is in RESERVATION_HELD or ORDER_PLACED state.

**Steps:**
1. In the session list, select the session.
2. Type: `Thanks, that's all I need.`
3. Click **Send**.

**Expected:**
- Agent replies with a closing message and calls no tool.
- The workflow advances to `closeStep` and calls `SessionEntity.close`.
- `SessionEntity` emits `SessionClosed`.
- Session status transitions to `CLOSED`.
- The session card in the live list shows status `CLOSED` in a muted-dark pill.
- The input bar is disabled for the closed session.
