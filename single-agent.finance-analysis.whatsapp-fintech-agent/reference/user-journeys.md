# User journeys — whatsapp-fintech-agent

## J1 — Balance query resolves without a transaction

**Preconditions:** Service running on declared port (`http://localhost:9589/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9589/` → App UI tab.
2. From the **Account** dropdown, pick any seeded account.
3. Click **Load example** and select the balance-query seed message.
4. Click **Send message**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted message text; any PII category chips show `phone` or similar if the seed message contains PII.
- Within 30 s the card reaches `RESPONSE_READY` and then immediately `EXECUTED`. The right pane shows: intent badge `BALANCE_QUERY`, the agent's `responseText` quoting the current balance, and a tool-calls entry for `get_balance`.
- No pending-transaction panel appears. No approve/reject buttons appear.

## J2 — Small fund transfer executes without HITL pause

**Preconditions:** Service running. Any model provider. The seeded small-transfer message amount is below $500 (the configured HITL threshold).

**Steps:**
1. From the Account dropdown, pick the account used as the transfer source.
2. Click **Load example** and select the small-transfer seed message ($50).
3. Click **Send message**.

**Expected:**
- The card reaches `SANITIZED` within 1 s. PII category chips show `phone` (the seed message contains a phone number).
- Within 30 s the card reaches `RESPONSE_READY` and immediately transitions to `EXECUTED` — no `AWAITING_APPROVAL` state appears in the card history.
- The right pane shows intent badge `FUND_TRANSFER`, the agent's `responseText`, a tool-calls entry for `transfer_funds` with amount `50.00`, and a pending-transaction card. Because the amount is below the threshold, the HITL gate was bypassed and the transaction executed automatically.

## J3 — Large fund transfer pauses for operator approval

**Preconditions:** Service running. Any model provider. The seeded large-transfer message amount is $1200 (above the $500 threshold).

**Steps:**
1. From the Account dropdown, pick the transfer-source account.
2. Click **Load example** and select the large-transfer seed message ($1200).
3. Click **Send message**.
4. Wait for the card to reach `AWAITING_APPROVAL` (the card border pulses orange).
5. Click **Approve**, enter an operator note in the prompt, and confirm.

**Expected:**
- The card transitions through `RECEIVED → SANITIZED → RESPONDING → RESPONSE_READY → AWAITING_APPROVAL`. At `AWAITING_APPROVAL`, the right pane shows the pending-transaction detail and the **Approve** / **Reject** buttons.
- After approval, the card transitions to `EXECUTED`. The right pane adds a HITL decision record showing outcome `APPROVED`, the operator ID entered, and the approval timestamp.
- The `POST /api/messages/{id}/approve` call returns `204`.

## J4 — Large fund transfer rejected by operator

**Preconditions:** Service running. Same setup as J3.

**Steps:**
1. Submit the large-transfer seed message (steps 1–3 from J3).
2. Wait for `AWAITING_APPROVAL`.
3. Click **Reject**, enter a rejection reason, and confirm.

**Expected:**
- The card transitions to `HITL_REJECTED`. The status pill shows red.
- The right pane shows the HITL decision record with outcome `REJECTED` and the operator's note.
- The `POST /api/messages/{id}/reject` call returns `204`.
- The pending-transaction card remains visible for audit, with the HITL rejection overlaid. No funds were moved.

## J5 — Before-tool-call guardrail blocks an invalid payment, agent retries

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `handle-customer-message.json` includes a MALFORMED entry with a destination `accountId` that is not in the seeded account registry.

**Steps:**
1. Submit messages until the 4th message (the mock selects the malformed entry on the first iteration of every 4th message by seed).
2. Watch the service log and the SSE stream in the browser dev tools (`/api/messages/sse`).

**Expected:**
- The first agent iteration issues a `transfer_funds` tool call with an invalid `toAccountId`.
- `PaymentGuardrail` rejects it. No funds-movement tool call completes with the invalid argument.
- The service log shows one `guardrail.reject` line with the structured-error code naming the failed check (`destination-account-not-found`).
- The agent loop retries on iteration 2 with a corrected tool call (or an `UNKNOWN`-intent response asking for clarification). The card transitions to `RESPONSE_READY` with a valid `AgentResponse`.

## J6 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider.

**Steps:**
1. Submit a message containing the literal strings `+44 7700 900123` (a test phone number) and `ACC-9988-7766` (a synthetic account number pattern).
2. Wait for `RESPONSE_READY`.
3. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).
4. Fetch `GET /api/messages/{id}` and read `inbound.rawText`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-PHONE]`, `[REDACTED-ACCOUNT-NUMBER]`. The raw strings do not appear.
- `inbound.rawText` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `phone`, `account-number` (at minimum).
