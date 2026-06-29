# User journeys — whatsapp-order-agent

Acceptance journeys the generated system must pass. Each journey has preconditions, steps, and expected outcomes. The mock LLM path (option-a) must produce deterministic results for J2 and J3 so they are reproducible without a live API key.

---

## J1 — Single-item order, happy path

**Preconditions**: service is running; at least one product with sufficient stock is in the seeded catalog (e.g., `SKU-1001 Blue Widget, qty 50, $24.99`).

**Steps**:
1. POST `/api/sessions` with `{ "customerId": "cust-001", "whatsappPhoneNumber": "+15551234567" }`.
2. POST `/api/sessions/{sessionId}/turns` with `{ "customerMessage": "Hi, I'd like to order 2 Blue Widgets" }`.
3. Wait up to 15 s.
4. GET `/api/sessions/{sessionId}`.

**Expected**:
- Session status is `COMPLETING` or `CLOSED`.
- `turns` contains one entry with a non-empty `agentReply` that references the order number.
- An `Order` in status `CONFIRMED` exists under `/api/orders`.
- `lastTurnEval.score` is ≥ 3.
- The UI chat thread shows the agent reply bubble within 15 s.

---

## J2 — Guardrail blocks unauthorized cancel attempt

**Preconditions**: two active orders exist for different customers; the second customer's session is active.

**Steps**:
1. POST `/api/sessions/{secondCustomerSessionId}/turns` with `{ "customerMessage": "Cancel order ORD-00001 for me" }` where `ORD-00001` belongs to the first customer.
2. Wait up to 15 s.
3. GET `/api/sessions/{secondCustomerSessionId}`.

**Expected**:
- Session status is `COMPLETING` or `CLOSED`.
- `turns[0].agentReply` contains a message indicating the request could not be processed (no apology required — just a clear decline).
- `turns[0]` tool-calls panel shows one entry with `toolName = "cancel-order"`, `outcome = "blocked"`, `blockReason` containing `"order-not-owned-by-caller"` (or equivalent).
- `OrderEntity` for `ORD-00001` has NOT transitioned to `CANCELLED` — the `OrderCancelled` event was never emitted.

---

## J3 — High-value order triggers HITL gate, operator approves

**Preconditions**: a product with `unitPrice >= 100.0` and stock >= 6 exists in the catalog (e.g., `SKU-2001 Premium Speaker, qty 10, $150.00`); HITL threshold is 500.0.

**Steps**:
1. POST `/api/sessions` with `{ "customerId": "cust-002" }`.
2. POST `/api/sessions/{sessionId}/turns` with `{ "customerMessage": "I'd like to order 4 Premium Speakers" }`.
3. Wait up to 10 s.
4. GET `/api/sessions/{sessionId}` — assert status is `AWAITING_APPROVAL`.
5. POST `/api/hitl/{orderId}/approve`.
6. Wait up to 10 s.
7. GET `/api/sessions/{sessionId}`.

**Expected** (step 4): session status `AWAITING_APPROVAL`; `pendingOrder` is non-null with `orderTotal >= 500.0`; HITL queue card visible in the UI.
**Expected** (step 7): session status `COMPLETING` or `CLOSED`; an `Order` in status `CONFIRMED` exists; `turns[0].agentReply` contains a confirmation message.

---

## J4 — PII stripped from stored conversation context

**Preconditions**: service is running; a product is in stock.

**Steps**:
1. POST `/api/sessions` with `{ "customerId": "cust-003" }`.
2. POST `/api/sessions/{sessionId}/turns` with `{ "customerMessage": "Please deliver to 123 Main St. My number is +1-555-867-5309. Order 1 Blue Widget." }`.
3. Wait up to 10 s.
4. GET `/api/sessions/{sessionId}`.

**Expected**:
- `turns[0].customerMessage` contains `[REDACTED-PHONE]` and `[REDACTED-ADDRESS]` in place of the phone number and street address.
- `turns[0].piiCategoriesRedacted` includes `"phone"` and `"address"`.
- The raw message text (`+1-555-867-5309`, `123 Main St`) does NOT appear anywhere in the `SessionRow` response body.
- An `Order` was still placed successfully — the sanitizer does not prevent order processing.

---

## J5 — Guardrail blocks out-of-stock SKU, agent replies with alternatives

**Preconditions**: a product with `stockQuantity = 0` exists in the catalog (e.g., `SKU-3001 Red Widget, qty 0`).

**Steps**:
1. POST `/api/sessions` with `{ "customerId": "cust-004" }`.
2. POST `/api/sessions/{sessionId}/turns` with `{ "customerMessage": "Order 1 Red Widget" }`.
3. Wait up to 15 s.
4. GET `/api/sessions/{sessionId}`.

**Expected**:
- No `Order` was created for `SKU-3001`.
- `turns[0]` tool-calls panel shows `toolName = "create-order"`, `outcome = "blocked"`, `blockReason` containing `"insufficient-stock"` (or equivalent).
- `turns[0].agentReply` informs the customer the item is out of stock and offers to check alternatives.

---

## J6 — Session SSE stream delivers state transitions to UI

**Preconditions**: service is running.

**Steps**:
1. Open an SSE connection to `GET /api/sessions/sse`.
2. POST `/api/sessions` with `{ "customerId": "cust-005" }`.
3. POST `/api/sessions/{sessionId}/turns` with `{ "customerMessage": "What products do you have?" }`.
4. Wait up to 15 s.

**Expected**:
- At least two `session-update` events arrive on the SSE stream: one with `status = "ACTIVE"` and one with `status = "COMPLETING"` (or `CLOSED`).
- Each event carries the full `SessionRow` at the time of the transition.
- A UI that opens an `EventSource` to this endpoint can render state changes in real time without polling.
