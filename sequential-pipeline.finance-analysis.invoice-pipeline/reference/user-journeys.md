# User journeys — invoice-processing

## J1 — Submit a standard invoice and get a posted journal entry

**Preconditions:** Service running on declared port (`http://localhost:9183/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded invoice `acme-standard-001` has total amount 3 200.00 USD — below the 10 000.00 threshold.

**Steps:**
1. Open `http://localhost:9183/` → App UI tab.
2. From the **Pick a seeded invoice** dropdown, select `Acme Corp — INV-2026-0421 (3 200.00 USD)`.
3. Click **Submit invoice**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `EXTRACTING` within 1 s more.
- Within ~20 s the card reaches `EXTRACTED`. The right pane shows the header table with `vendorEmail = "[REDACTED]"` and `vendorPhone = "[REDACTED]"`, plus a line-item table with ≥ 2 rows. The green PII-redacted badge is visible.
- Within ~20 s more the card reaches `VALIDATED`. The right pane shows the validated line-item table with GL account codes and `balanceOk = true` for every row; the balance summary confirms debitTotal equals creditTotal.
- Because the total is 3 200.00 (below threshold), no approval panel appears. The card advances directly to `POSTING`, then `POSTED`, then `EVALUATED` within 1 s of `POSTED`.
- The right pane shows the journal entry table with debit lines for each expense GL account and one credit line to Accounts Payable, totalling 3 200.00 on each side. The eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s.

## J2 — High-value invoice pauses for reviewer approval

**Preconditions:** Service running. The seeded invoice `bigco-highvalue-002` has total amount 47 500.00 USD — above the 10 000.00 threshold.

**Steps:**
1. From the **Pick a seeded invoice** dropdown, select `BigCo Inc — INV-2026-0088 (47 500.00 USD)`.
2. Click **Submit invoice**.
3. Wait until the card shows `APPROVAL_REQUESTED`.
4. Open the right pane for that invoice. Click **Approve** and supply a reviewer note.

**Expected:**
- The card transitions through `EXTRACTING → EXTRACTED → VALIDATING → VALIDATED` as in J1.
- After `VALIDATED`, the card transitions to `APPROVAL_REQUESTED` (orange status pill and "Awaiting approval" badge). The Approve / Deny button pair appears in the right pane.
- While the card is in `APPROVAL_REQUESTED`, the workflow is suspended — no further SSE events arrive for this invoice.
- After clicking Approve, the entity emits `ApprovalGranted`. Within ~1 s the card transitions to `APPROVED`, then `POSTING`, then `POSTED`, then `EVALUATED`.
- The final journal entry totals 47 500.00 on both debit and credit sides. The eval score chip shows 5/5.

## J3 — Vendor PII is never visible in the entity or API

**Preconditions:** Service running. Any invoice (standard or high-value).

**Steps:**
1. Submit the seeded invoice `acme-standard-001`.
2. Wait for `EXTRACTED`.
3. Call `GET /api/invoices/{id}` from the browser dev tools or curl.
4. Inspect the response body and the SSE stream events for this invoice.

**Expected:**
- In the `extracted.header` of the API response, `vendorEmail` is `"[REDACTED]"` and `vendorPhone` is `"[REDACTED]"`.
- The `sanitization.redactedFields` array contains `["vendorEmail", "vendorPhone"]`. The `sanitization` object contains no original values — only field names.
- In the SSE stream, the `EXTRACTED` event carries the same sanitized header; no raw contact data appears in any event payload.
- The App UI header table shows the same `[REDACTED]` values. The PII-redacted badge is visible on the right pane.
- No log line at any level echoes the original `vendorEmail` or `vendorPhone` values after `extractStep` begins.

## J4 — Unbalanced journal entry flags eval score ≤ 2

**Preconditions:** Mock LLM mode. The mock's `post-journal-entry.json` includes one entry where the sum of debit lines does not equal the sum of credit lines (e.g., debits total 3 200.00, credits total 2 800.00).

**Steps:**
1. Submit any seeded invoice six times. (The unbalanced entry is selected once in every six runs by the mock's `seedFor(invoiceId)` modulo.)
2. Watch the live list until a card border highlights red.

**Expected:**
- The flagged invoice lands well-formed (the guardrail checks phase order, not journal balance).
- The eval score chip shows **2** (or lower) and the rationale reads something like: *"Journal balance failed: debit total 3 200.00 does not equal credit total 2 800.00 — shortfall 400.00."*
- The card's border highlights red. The finance reviewer knows to inspect this posting before any downstream cash disbursement.
- The other five invoices in the run scored ≥ 4.

## J5 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM (`model-provider = mock`). The mock's `extract-invoice.json` includes one entry whose `tool_calls` array starts with `buildJournalEntry` (a POST-phase tool called during EXTRACT).

**Steps:**
1. Submit any seeded invoice three times in a row.
2. Watch the third submission's lifecycle in the network panel (`/api/invoices/sse`).

**Expected:**
- On the third submission's `extractStep`, the agent's first iteration calls `buildJournalEntry`. `PhaseGuardrail` rejects it; a `PhaseGuardrailRejected{phase: "EXTRACT", tool: "buildJournalEntry", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `PostTools.buildJournalEntry` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal extract sequence (`parseHeader` + `parseLineItems`). The card eventually reaches `EVALUATED` as in J1.

## J6 — Reviewer denies a high-value invoice

**Preconditions:** Service running. The seeded invoice `bigco-highvalue-002` (47 500.00 USD) is in `APPROVAL_REQUESTED`.

**Steps:**
1. Submit `bigco-highvalue-002` and wait for `APPROVAL_REQUESTED`.
2. In the right pane, click **Deny** and enter a reviewer note: `"PO not found — returning to vendor."`

**Expected:**
- The entity emits `ApprovalDenied{reviewerNote}`. The card transitions to `DENIED` (terminal; muted pill).
- No `POSTING`, `POSTED`, or `EVALUATED` events appear. The journal entry panel is empty.
- The workflow ends without calling `InvoiceAgent` for the POST task. The eval panel is absent (no posting, no score).
- The card remains in the live list as `DENIED` with the full extracted and validated data visible for audit purposes.
