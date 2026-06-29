# User journeys — airline-cs

## J1 — Seat change with HITL confirmation

**Preconditions:** Service running on declared port (`http://localhost:9640/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. Seeded booking `ABC123` exists with `upgradeEligible: true`.

**Steps:**
1. Open `http://localhost:9640/` → App UI tab.
2. From the **Request type** dropdown, pick `Seat change`.
3. Click **Load seeded example** to fill booking ref (`ABC123`) and message textarea.
4. Click **Submit request**.
5. Watch the card in the live list.
6. When the card reaches `AWAITING_CONFIRMATION`, click **Confirm** in the right pane.

**Expected:**
- The card appears in `RECEIVED` within 1 s, transitions to `SANITIZED` within 1 s (PII category chip shows `email` is redacted from the message).
- Within ~30 s the card reaches `AWAITING_CONFIRMATION`. The right pane shows the proposed change ("Move from seat 22C to 14A on flight AA200") and the Confirm / Cancel buttons.
- After clicking Confirm, the card transitions back to `PROCESSING` within 1 s, then reaches `COMPLETED` within ~15 s.
- The tool call log shows `searchBooking → requestConfirmation → modifyReservation` in order.
- The outcome badge shows `RESOLVED` and the summary reads "Seat changed from 22C to 14A on flight AA200."

## J2 — Guardrail blocks modification without confirmation token

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `handle-service-request.json` includes an entry where the agent calls `modifyReservation` before calling `requestConfirmation`.

**Steps:**
1. Submit the seat-change seeded example four times in a row (J1 steps × 4, without clicking Confirm — let the workflow progress naturally).
2. On the fourth submission, watch the service log and the tool call log in the UI.

**Expected:**
- The fourth submission's first agent iteration calls `modifyReservation` without a `confirmationToken`.
- `ModificationGuardrail` returns `MISSING_CONFIRMATION_TOKEN` and the call to `BookingService` never executes — there is no `ModificationResult` in the tool call log for that iteration.
- The agent loop receives the rejection, calls `requestConfirmation` on its next iteration, and transitions the entity to `AWAITING_CONFIRMATION`.
- After the customer confirms, the second `modifyReservation` carries the token and succeeds.
- The service log shows one `guardrail.reject` line with reason `MISSING_CONFIRMATION_TOKEN`.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit a custom message: "Change my seat on ABC123, frequent flyer number FFP-7654321, contact jane@test.com, phone +1-555-9876."
2. Wait for `AWAITING_CONFIRMATION` or `COMPLETED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).
4. Fetch `GET /api/requests/{id}` and read `request.rawMessage`.

**Expected:**
- The logged LLM call body contains only redacted tokens: `[REDACTED-FFP]`, `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`. The raw strings do not appear.
- `request.rawMessage` in the JSON still contains `FFP-7654321`, `jane@test.com`, and `+1-555-9876` — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `ffp`, `email`, `phone` (at minimum).

## J4 — Complaint filed without HITL gate

**Preconditions:** Service running. Any model provider. Seeded booking `MX789` corresponds to a missed-connection scenario.

**Steps:**
1. From the **Request type** dropdown, pick `Complaint`.
2. Click **Load seeded example** to fill booking ref (`MX789`) and the complaint message.
3. Click **Submit request**.
4. Wait for `COMPLETED`.

**Expected:**
- The card reaches `COMPLETED` without ever entering `AWAITING_CONFIRMATION` — the HITL gate is not triggered for complaint-only flows.
- The tool call log shows only `fileComplaint`. No `searchBooking`, no `requestConfirmation`, no `modifyReservation`.
- The outcome section shows a non-null `caseNumber` badge (e.g., `CASE-00412`).

## J5 — Customer cancels a proposed change

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seat-change seeded example.
2. When the card reaches `AWAITING_CONFIRMATION`, click **Cancel** in the right pane.

**Expected:**
- The entity transitions to `CANCELLED` within 1 s.
- The outcome badge shows `CANCELLED`. The summary reads "Customer cancelled the proposed seat change."
- No `modifyReservation` call appears in the tool call log — the booking is unchanged.
- `BookingService` was never called with a write operation.

## J6 — Non-modifiable booking returns NEEDS_FOLLOWUP

**Preconditions:** Service running. Seeded booking `LK456` has `upgradeEligible: false` and fare class `W` (non-refundable).

**Steps:**
1. Submit a seat-change request for booking `LK456`.
2. Wait for `COMPLETED`.

**Expected:**
- The agent calls `searchBooking("LK456")`, reads `upgradeEligible: false`, and does NOT call `requestConfirmation` or `modifyReservation`.
- The outcome status is `NEEDS_FOLLOWUP`. The summary explains why (e.g., "Seat upgrades are not permitted on W-fare tickets. Contact the fare desk for options.").
- The HITL gate is never triggered — no `AWAITING_CONFIRMATION` state is reached.
