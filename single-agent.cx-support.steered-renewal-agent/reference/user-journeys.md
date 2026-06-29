# User journeys — steered-renewal-agent

## J1 — Eligible patron renews an available book

**Preconditions:** Service running on declared port (`http://localhost:9354/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9354/` → App UI tab.
2. From the **Patron** dropdown, pick `Alex Rivera` (STANDARD tier, zero fines).
3. From the **Loan** dropdown, pick `Designing Data-Intensive Applications` (BOOK, 0 prior renewals, max 3).
4. Click **Request renewal**.

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `ENRICHED` within 1 s. The right pane shows Alex Rivera's patron summary (fines: $0.00) and loan detail.
- Within 30 s the card reaches `DECISION_RECORDED`. The right pane shows outcome badge `APPROVED`, a reason sentence, and a `newDueDate` approximately 14 days in the future.
- Within 1 s of `DECISION_RECORDED`, the card reaches `COMPLETED` and shows the notification record (channel: in-app, message confirming the new due date).

## J2 — Guardrail blocks a policy-violating approval

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `renew-loan.json` includes a deliberately policy-violating entry (APPROVED outcome for a patron above the fine threshold).

**Steps:**
1. From the **Patron** dropdown, pick `Sam Torres` (STANDARD tier, $20.00 outstanding fines — above the $10.00 threshold).
2. From the **Loan** dropdown, pick any eligible loan.
3. Submit the renewal request three times in a row.
4. On the third submission, watch the SSE feed (`/api/renewals/sse`) in the browser network panel.

**Expected:**
- The third submission's first agent iteration produces an APPROVED decision — a policy violation.
- `PolicyEnforcer` rejects it. The APPROVED decision NEVER reaches `LoanEntity.recordDecision` — there is no `DecisionRecorded` event with the APPROVED payload.
- The agent loop retries on iteration 2 (or 3) and produces a DENIED decision citing the fine balance. The card reaches `COMPLETED` with outcome DENIED.
- The service log shows one `guardrail.reject` line with error code `fine-threshold-exceeded`.

## J3 — Loan at maximum renewal count is denied

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Patron** dropdown, pick `Morgan Chen` (PREMIUM tier, $4.50 fines — below threshold).
2. From the **Loan** dropdown, pick the loan entry labelled `(max renewals reached)` — this loan has `priorRenewalCount == maxRenewalsAllowed`.
3. Click **Request renewal**.

**Expected:**
- The card reaches `DECISION_RECORDED` with outcome `DENIED`.
- The reason sentence references the maximum renewal count (e.g., "This item has reached its maximum renewal limit of 3.").
- No `newDueDate` is present on the decision.
- The card reaches `COMPLETED` with a denial notification.

## J4 — Live SSE updates multiple browser tabs simultaneously

**Preconditions:** Service running. Two or more browser tabs open to `http://localhost:9354/`.

**Steps:**
1. In Tab A, submit a renewal request for any eligible patron and loan.
2. While the renewal is processing (before COMPLETED), open Tab B and navigate to the same URL.
3. Observe both tabs.

**Expected:**
- Tab A shows the renewal card updating through each state transition in real time.
- Tab B, opened mid-flight, receives the full current row on its SSE connection and displays the renewal at whatever state it is currently in — no page reload or manual refresh required.
- Both tabs show `COMPLETED` once the workflow finishes, without either tab refreshing.

## J5 — FAILED renewal preserves partial state

**Preconditions:** Mock LLM mode. The mock is configured to exhaust all three iterations with policy-violating responses for a specific patronId (edit `renew-loan.json` to remove all valid entries for that patron).

**Steps:**
1. Submit a renewal for the patron whose mock entries are all policy-violating.
2. Wait for the card to stop updating.

**Expected:**
- All three agent iterations produce policy-violating decisions. `PolicyEnforcer` rejects each one.
- After three rejections, `decideStep`'s recovery path fires and calls `RenewalWorkflow::error`.
- The card transitions to `FAILED`.
- Fetching `GET /api/renewals/{id}` returns the full Renewal record with `patron` and `loan` populated (enrichStep completed), `decision` as null, `notification` as null, and `status = FAILED`. The partial state is preserved for administrator review.
