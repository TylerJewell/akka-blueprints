# User journeys — finance-close-reconciler

## J1 — Submit an IFRS balance and receive an attested report

**Preconditions:** Service running on declared port (`http://localhost:9365/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9365/` → App UI tab.
2. From the **Rule set** dropdown, pick `IFRS-operating (8 rules)`.
3. Click **Load seeded example** to fill the period label and the trial-balance textarea.
4. Enter `accountant-7` in the **Submitted by** field.
5. Click **Submit for reconciliation**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the masked balance CSV; the masked-field-type chips show `margin-pct` and `entity-code` (at minimum).
- Within 30 s the card reaches `REPORT_RECORDED`. The right pane shows: a report badge (IN_BALANCE / HAS_VARIANCES / INCOMPLETE), the summary paragraph, and a finding row for *each of the 8 submitted rules*. Every finding has non-empty `accountPair`, `expectedBalance`, `actualBalance`, and `variance` values.
- The card moves to `AWAITING_SIGNOFF`. The **Approve** / **Reject** button pair is visible in the detail pane.
- The accountant clicks **Approve** (leaving the note field empty). The card transitions to `ATTESTED`. The attestation chip shows `complete: true` and the timestamp.

## J2 — Guardrail blocks an invalid GL write proposal

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `reconcile-period.json` includes an entry with a proposed write to a non-existent account code.

**Steps:**
1. Submit the IFRS-operating seed balance three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/periods/sse`).

**Expected:**
- The third submission's first agent iteration proposes a GL write to an account code not in the submitted rules.
- `WriteGuardrail` rejects it. The invalid write proposal NEVER lands in `PeriodEntity` — there is no `ReportRecorded` event referencing the bad account code.
- The agent loop retries on iteration 2 and produces a valid write proposal with a corrected account code. The card transitions to `REPORT_RECORDED` with a report whose `proposedGlWrites` are all valid.
- The service log shows one `guardrail.reject` line per rejected tool call with the structured-error code naming which check failed (`account-not-found` or `unbalanced-entry`).

## J3 — Accountant rejects the report with a note

**Preconditions:** Service running. Any model provider. A period in `AWAITING_SIGNOFF` state (complete J1 up to step 5).

**Steps:**
1. In the selected-period detail pane, find the sign-off section.
2. Enter `controller-3` in the approver field.
3. Enter `Variance in account 2200 needs further investigation before approval.` in the note textarea.
4. Click **Reject**.

**Expected:**
- The card transitions to `REJECTED` within 1 s. The rejection note appears in muted text in the detail pane.
- The attestation section is absent — no `AttestationCompleted` event was emitted.
- `GET /api/periods/{id}` returns `status: "REJECTED"` and `signOff.decision: "REJECTED"`.

## J4 — Confidential fields never reach the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a trial balance containing the literal columns `GROSS_MARGIN_PCT=0.32`, `ENTITY_CODE_EU=EUROPLANT`, and `INTERNAL_COST_CTR=CC-9901`.
2. Wait for `REPORT_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/periods/{id}` and read `submission.rawBalance`.

**Expected:**
- The logged LLM call body contains the masked forms only: `[MASKED-MARGIN-PCT]`, `[MASKED-ENTITY-CODE]`, `[MASKED-INTERNAL-CODE]`. The raw values `0.32`, `EUROPLANT`, and `CC-9901` do not appear.
- `submission.rawBalance` in the JSON still contains the raw CSV columns — the audit log preserves them.
- `sanitized.maskedFieldTypes` lists `margin-pct`, `entity-code`, `internal-code` (at minimum).

## J5 — Rule completeness enforced by guardrail

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the GAAP-manufacturing seed balance with the `GAAP-manufacturing (12 rules)` rule set.
2. Wait for `ATTESTED` (including sign-off approval).

**Expected:**
- The report's `findings` array has exactly 12 entries, one per submitted rule. No silent omissions.
- Each `findings[i].ruleId` matches one of the 12 submitted `ruleId` values, one-to-one.
- The CI attestation gate (`AttestationGateTest`) passes for the corresponding fixture, confirming the event chain is complete.

## J6 — Sign-off timeout transitions to FAILED

**Preconditions:** Service running with `awaitSignOffStep` timeout overridden to 10 s via `application.conf` for this test scenario. A period in `AWAITING_SIGNOFF` state.

**Steps:**
1. Submit any seed balance and wait for `AWAITING_SIGNOFF`.
2. Do not call the sign-off endpoint. Wait 15 s.

**Expected:**
- The card transitions to `FAILED` with reason `sign-off-timeout`.
- No `AttestationCompleted` event was emitted.
- The submission and report data are preserved on the entity — the accountant can inspect the partial state via `GET /api/periods/{id}`.
