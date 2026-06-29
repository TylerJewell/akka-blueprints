# User journeys — antom-payment

## J1 — Submit a sub-threshold INITIATE and reach SETTLED

**Preconditions:** Service running on declared port (`http://localhost:9678/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9678/` → App UI tab.
2. Click **Load seeded example** and pick the `INITIATE USD sub-threshold` entry (500 USD, recipient `antom-merchant-0042`, method BANK_TRANSFER).
3. Click **Submit instruction**.

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `AUTHORIZED` within 1 s. The right-pane detail shows all instruction fields.
- The card transitions to `EXECUTING` (skipping AWAITING_APPROVAL because the amount is below the threshold) within 1 s.
- Within 30 s the card reaches `SETTLED`. The right pane shows: the Antom API response block (non-null `transactionId`, `antomStatus = "SUCCESS"`, `settledAmountMinorUnits`, `feeMinorUnits`), and the agent summary paragraph referencing the transaction id.
- `finishedAt` is non-null.

## J2 — High-value INITIATE pauses for operator approval

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Load the `INITIATE EUR high-value` seeded example (15 000 EUR, above the 10 000 threshold).
2. Click **Submit instruction**.
3. Observe the card and the right-pane approval controls.
4. Click **Approve** in the right pane.

**Expected:**
- The card transitions: `REQUESTED → AUTHORIZED → AWAITING_APPROVAL`. The amber "Approval pending" chip appears.
- The card does not advance to `EXECUTING` until Approve is clicked.
- After clicking Approve, the card transitions to `EXECUTING` then `SETTLED` within 30 s.
- If **Deny** is clicked instead, the card transitions to `REJECTED` and `finishedAt` is set. No agent call is made.

## J3 — Fraud signal halts an executing payment

**Preconditions:** Service running with the `AntomApiSimulator` active. The simulator emits `FraudSignalDetected` for payments whose `paymentId` hash mod 7 == 0.

**Steps:**
1. Submit instructions one by one until the 7th payment (or load the seeded `velocity-breach` example whose paymentId is pre-seeded to hit the mod-7 path).
2. Watch the card in the live list.

**Expected:**
- The card transitions through `REQUESTED → AUTHORIZED → EXECUTING`.
- While in `EXECUTING`, a `FraudSignalDetected` event fires. The card immediately transitions to `HALTED`.
- The right pane shows the fraud signal section: signal type `velocity-breach`, detail string, detected at timestamp. The card border highlights red.
- No Antom API response is recorded — `result` is null. The partial execution state (instruction fields) is visible for audit.
- No further agent tool calls appear in the service log after the halt.

## J4 — Blocked recipient triggers guardrail rejection

**Preconditions:** Service running. The `blocked-recipients.json` seed file contains `antom-merchant-BLOCKED`.

**Steps:**
1. In the Instruction panel, enter type INITIATE, currency USD, amount 100, recipient ID `antom-merchant-BLOCKED`, method WALLET, submitted by `operator-test`.
2. Click **Submit instruction**.

**Expected:**
- The card transitions to `REQUESTED` then immediately to `REJECTED`.
- The right pane shows the rejection reason: `authorization-failure: recipientId is on the blocked-recipient registry`.
- The service log shows one `guardrail.reject` line naming the failed check (`blocked-recipient`).
- No `ExecutionStarted` event is recorded; no agent or Antom API call is made.

## J5 — QUERY instruction returns status without settlement

**Preconditions:** A prior INITIATE payment with known `paymentId` has reached `SETTLED` and has a `transactionId`. The simulator assigns deterministic transactionIds via `seedFor(paymentId)`.

**Steps:**
1. Load the `QUERY` seeded example, which references the transactionId from the first seeded INITIATE.
2. Click **Submit instruction**.

**Expected:**
- The card transitions: `REQUESTED → AUTHORIZED → EXECUTING → QUERIED`.
- The right pane shows the Antom API response with `antomStatus = "PENDING"` or `"SUCCESS"` (depending on the simulator seed) and the transactionId.
- `outcome = "queried"` in the result.
- `finishedAt` is set; the card is terminal.

## J6 — REFUND instruction completes successfully

**Preconditions:** A prior INITIATE payment has reached `SETTLED` with a known `transactionId`.

**Steps:**
1. Load the `REFUND` seeded example (references the settled transactionId, partial amount 200 USD).
2. Click **Submit instruction**.

**Expected:**
- The card transitions: `REQUESTED → AUTHORIZED → EXECUTING → SETTLED` (outcome = `refunded`).
- The Antom API response block shows `antomStatus = "SUCCESS"` and `settledAmountMinorUnits` matching the refund amount.
- The agent summary references the refund and the original transactionId.
