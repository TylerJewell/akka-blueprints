# User journeys — retail-customer-service

## J1 — Product inquiry with catalog lookup

**Preconditions:** Service running on `http://localhost:9523/`; a valid model-provider API key set, or the mock LLM selected at scaffold time. Seeded product catalog loaded (12 products in `products.jsonl`).

**Steps:**
1. Open `http://localhost:9523/` → App UI tab.
2. In the **Scenario** dropdown, pick `Product inquiry`. Click **Load scenario** to pre-fill Customer ID and Message: "Do you have any low-light succulents that are safe for cats?"
3. Click **Send**.

**Expected:**
- A new session card appears in the left panel with status `OPEN` within 1 s.
- Within 30 s, the status transitions to `ACTIVE` and the conversation thread shows the agent's reply.
- The reply names at least one plant from the seeded catalog (Haworthia fasciata is marked `petFriendly: true` in the seed data and tolerates indirect light).
- The reply includes a price grounded in the catalog (`priceUsd` field). No price is quoted more than 10% below the catalog value.
- No competitor brand names appear in the reply.
- The turn status indicator in the session card shows a green dot (REPLIED).

## J2 — Address change on a PENDING order (guardrail allows)

**Preconditions:** Service running. Seeded order `ORD-1042` has `status = PENDING`.

**Steps:**
1. In the **Scenario** dropdown, pick `Address change`. Click **Load scenario** (pre-fills Customer ID, Order ID `ORD-1042`, Message: "Can you change my delivery address to 789 Maple Ave, Chicago, IL 60601?").
2. Click **Send**.

**Expected:**
- `OrderModificationGuardrail` fetches order `ORD-1042`, finds status `PENDING`, and allows the `updateOrderAddress` tool call.
- The agent's reply confirms the address change.
- An **Order modified** chip (teal) appears below the reply showing "Address updated — ORD-1042".
- `GET /api/orders/ORD-1042` returns the order with `shippingAddress = "789 Maple Ave, Chicago, IL 60601"` and one entry in `modifications` with `modificationType = "update-address"`.
- The `OrderAddressUpdated` event appears in `OrderEntity`'s event log.

## J3 — Cancellation attempt on a SHIPPED order (guardrail blocks)

**Preconditions:** Service running. Seeded order `ORD-1055` has `status = SHIPPED`.

**Steps:**
1. In the **Scenario** dropdown, pick `Order cancellation`. Click **Load scenario** (pre-fills Order ID `ORD-1055`, Message: "Please cancel my order ORD-1055.").
2. Click **Send**.

**Expected:**
- `OrderModificationGuardrail` fetches order `ORD-1055`, finds status `SHIPPED`, and rejects the `cancelOrder` tool call with `ORDER_NOT_MODIFIABLE current status: SHIPPED`.
- The agent incorporates the rejection into its reply, explaining that the order has shipped and cannot be cancelled, and offers a return option.
- A **Guardrail blocked** chip (amber) appears below the reply showing "ORDER_NOT_MODIFIABLE".
- No `OrderCancelled` event is emitted. `GET /api/orders/ORD-1055` still shows `status = SHIPPED`.
- The service log contains exactly one `guardrail.reject` line for this turn naming `ORDER_NOT_MODIFIABLE`.

## J4 — Before-agent-response blocks a competitor mention

**Preconditions:** Mock LLM mode. The mock's `handle-customer-turn.json` includes one entry whose `message` field contains the fictional brand name "PlantWorld". This entry is selected on the first iteration of a specific session seed.

**Steps:**
1. Submit a product-inquiry message with a customer ID that causes the mock's seed selection to land on the competitor-mention entry (documented in the mock's seed mapping in `MockModelProvider.java`).
2. Wait for `ACTIVE` status.

**Expected:**
- The first agent iteration produces a reply containing "PlantWorld".
- `ReplyPolicyGuardrail` rejects it with policy code `COMPETITOR_MENTION`.
- The agent loop retries on iteration 2 and produces a compliant reply with no competitor names.
- The session's conversation thread shows only the compliant reply — the rejected draft never appears.
- The service log shows one `guardrail.reject` line for `COMPETITOR_MENTION` on iteration 1, and a `guardrail.accept` line on iteration 2.

## J5 — Multi-turn conversation maintains context

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Send a first message: "What's in stock for outdoor ferns?"
2. When the reply arrives, send a follow-up: "How much water does the Boston Fern need?"
3. When the second reply arrives, send a third: "Great — add two to order ORD-1042."

**Expected:**
- Each turn is recorded as a separate `ConversationTurn` entry on `SessionEntity`.
- The second reply references the Boston Fern specifically (carrying context from the first turn), not a generic fern.
- The third message's `orderChange` (if the agent elects to apply it) references `ORD-1042` with `changeType = "update-quantity"`, and the before-tool-call guardrail checks the order's status before allowing the quantity update.
- `GET /api/sessions/{id}` returns all three turns in order, each with its own `turnId`, `startedAt`, `completedAt`, and reply.
