# User journeys — supplier-comms-monitor

## J1 — PO arrives and outreach is drafted

**Preconditions:** Service running on declared port; valid model-provider API key set; `PoOrderPoller` enabled.

**Steps:**
1. Open `http://localhost:9831/` → App UI tab.
2. Wait up to 20 s for the first simulated PO feed event.

**Expected:**
- PO appears with status OPENED, then within 1 s transitions to RISK_ASSESSED with a risk tier chip (ON_TRACK / AT_RISK / CRITICAL).
- For AT_RISK or CRITICAL POs: within 30 s the status reaches AWAITING_BUYER_APPROVAL with a drafted outreach visible in the detail pane.
- `requiresBuyerApproval` is `true` for CRITICAL POs and false for routine AT_RISK acknowledgment requests.

## J2 — Buyer approves outreach

**Preconditions:** A PO in AWAITING_BUYER_APPROVAL.

**Steps:**
1. Select the PO in the list.
2. (Optionally) edit the outreach draft body.
3. Click Approve.

**Expected:**
- Status transitions to OUTREACH_SENT immediately. The detail pane shows `buyerDecision.approved = true` and `decidedBy` is set from the UI's session id.
- An `OutreachSent` event is written to the entity; no actual network call leaves the process (the `sendSupplierEmail` tool is a no-op stub).
- The guardrail pre-call check passes (entity state satisfies send eligibility).
- The card moves to its terminal styling.

## J3 — Buyer escalates a critical PO

**Preconditions:** A PO in AWAITING_BUYER_APPROVAL with tier CRITICAL.

**Steps:**
1. Click Escalate. An escalation-note textarea opens.
2. Type a note; click Confirm.

**Expected:**
- Status transitions to PROCUREMENT_REVIEW. `buyerDecision.approved = false`, `escalationNote` is populated.
- The note is visible on the card in the detail pane.

## J4 — ON_TRACK PO auto-confirms without buyer interaction

**Preconditions:** The simulated feed contains a PO seeded with comfortable lead time.

**Steps:**
1. Wait for the ON_TRACK PO to appear in the board.

**Expected:**
- The PO's status transitions OPENED → RISK_ASSESSED → ON_TRACK_CONFIRMED without entering AWAITING_BUYER_APPROVAL.
- No Approve or Escalate action is required or available for ON_TRACK POs.

## J5 — Accuracy eval score appears on a closed PO

**Preconditions:** At least one CLOSED PO exists without an `accuracyScore`. The `AccuracyEvalRunner` schedule reduced for test (`EVAL_RUNNER_SECONDS=120`).

**Steps:**
1. Approve an outreach; wait for the PO to be marked CLOSED in the simulator.
2. Wait 120 s.

**Expected:**
- The card shows a `predictionCorrect` chip and the `evalRationale` is visible in the detail pane.
- The PO record in `GET /api/orders/{id}` includes `accuracyScore` populated.

## J6 — Guardrail blocks a send in an ineligible state

**Preconditions:** Service running with debug logging (`LOG_LEVEL=DEBUG`); a PO still in OUTREACH_DRAFTED (not yet AWAITING_BUYER_APPROVAL).

**Steps:**
1. Trigger the `sendSupplierEmail` tool stub directly for a PO where buyer approval has not been recorded.

**Expected:**
- The guardrail rejects the call with a structured error naming the blocking condition.
- The PO status does not change.
- The rejection is logged; no outbound message is produced.
