# User journeys — customer-service-tool-agent

## J1 — Order status query resolves in one turn

**Preconditions:** Service running on declared port (`http://localhost:9930/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. Seeded order `ORD-5501` exists for customer `CUST-1042`.

**Steps:**
1. Open `http://localhost:9930/` → App UI tab.
2. Select customer `CUST-1042` from the Customer dropdown.
3. Select scenario `Order status` (message pre-fills to "Where is my order ORD-5501?").
4. Click **Send**.

**Expected:**
- The new session card appears in the live list with status `OPEN` within 1 s, then transitions to `ACTIVE`.
- Within 30 s the card reaches `REPLIED`. The conversation thread shows:
  - The customer message.
  - A tool-calls row: `lookupOrder | ALLOWED | { status: SHIPPED, carrier: FedEx, ... }`.
  - The agent reply naming the shipping status, carrier, and estimated delivery date — sourced from the tool result.
- `lastScreened.piiCategoriesFound` is empty (no PII in this reply).

## J2 — Refund above threshold triggers escalation

**Preconditions:** Service running. Seeded order `ORD-5501` exists with total value $420.00. Auto-approve limit is $250.00 (default).

**Steps:**
1. Select customer `CUST-1042`, scenario `Refund request` (message pre-fills to "I want a refund for my order ORD-5501").
2. Click **Send**.

**Expected:**
- The session transitions to `ACTIVE`, then within 30 s to `ESCALATED`.
- The conversation thread shows:
  - A tool-calls table with two rows:
    - `processRefund | BLOCKED | blockReason: "Refund amount 420.00 exceeds auto-approve limit 250.0; escalate to human operator."`
    - `openEscalation | ALLOWED | { ticketId: "ESC-9910", status: "OPEN" }`
  - The agent reply naming the escalation ticket id and expected resolution time.
- The session card shows the orange `ESCALATED` badge with the ticket id `ESC-9910`.
- No `VerdictRecorded`-equivalent event carries the blocked refund amount as an approved transaction.

## J3 — Reply guardrail blocks a PAN in the draft and forces a retry

**Preconditions:** Service running with the mock LLM selected. The mock `handle-support-query.json` contains one entry whose `replyText` includes a 16-digit card-number pattern. The mock selects this entry on the first iteration of every 4th session.

**Steps:**
1. Start 4 sessions in sequence (J1 steps × 4, each a fresh session).
2. On the 4th session, watch the network panel (`/api/conversations/sse`) during the `ACTIVE` phase.

**Expected:**
- The 4th session's first agent iteration produces a reply with a card-number pattern.
- `ReplyGuardrail` rejects it. The malformed reply NEVER appears in `ConversationEntity` — there is no `AgentReplied` event with the raw PAN.
- The agent loop retries on iteration 2 and produces a clean reply. The session transitions to `REPLIED` with a reply that passes all four guardrail checks.
- The service log shows one `guardrail.reject` line naming the `pii-pan-detected` check code.

## J4 — PII is redacted in the screened log but preserved for audit

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Start a session for customer `CUST-1042` with a custom message: "Please update my email to jane.doe@acme.com and confirm my account number 4111111111111111."
2. Wait for `REPLIED`.
3. Inspect the service log for the `ReplyScreened` event payload.
4. Fetch `GET /api/conversations/{id}` and read `turns[0].reply.replyText`.

**Expected:**
- The `ReplyScreened` event's `redactedReplyText` in the log contains `[REDACTED-EMAIL]` and `[REDACTED-PCN]` — not the raw strings.
- `piiCategoriesFound` lists `email` and `payment-card-number`.
- `turns[0].reply.replyText` in the raw entity record (the JSON response) still contains the original reply text — the audit trail retains the unredacted form.
- The UI displays the screened form from `lastScreened.redactedReplyText`.

## J5 — Multi-turn follow-up continues the same session

**Preconditions:** Service running. Any model provider. One session already in `REPLIED` state.

**Steps:**
1. In the right-pane thread of a `REPLIED` session, type a follow-up message: "What is the expected delivery date?"
2. Click the inline Send button.

**Expected:**
- The session transitions back to `ACTIVE`, then within 30 s to `REPLIED` again.
- The conversation thread shows the original turn followed by the new turn.
- The agent's second reply cites the order record from the same tool call (it re-calls `lookupOrder` or references the prior turn's result from the history context).
- The session status stays in `REPLIED` (not re-opened as a new session).

## J6 — Inventory check returns stock level and agent replies correctly

**Preconditions:** Service running. Seeded product `SKU-APX-100` (Apex Pro Headset) has 12 units in stock.

**Steps:**
1. Select customer `CUST-1001`, scenario `Inventory check` (message: "Is the Apex Pro headset in stock?").
2. Click **Send**.

**Expected:**
- Tool-calls row: `checkInventory | ALLOWED | { sku: "SKU-APX-100", stockLevel: 12, available: true }`.
- Agent reply confirms the item is in stock and may include a suggested order action.
- `ReplyGuardrail` passes (no PAN, no SSN, at least one tool call, within 1000 chars).
- `piiCategoriesFound` is empty.
