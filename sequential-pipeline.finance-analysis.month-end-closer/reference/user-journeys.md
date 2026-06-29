# User journeys — month-end-closer

## J1 — Submit a close run and complete with two approvals

**Preconditions:** Service running on declared port (`http://localhost:9754/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded entity+period `ACME-US / 2026-05` has a matching `src/main/resources/sample-data/ledger/ACME-US-2026-05.json` file.

**Steps:**
1. Open `http://localhost:9754/` → App UI tab.
2. From the entity dropdown, pick `ACME-US`. From the period dropdown, pick `2026-05`.
3. Click **Start close run**.
4. Wait for the card to reach `AWAITING_GATHER_APPROVAL`. Review the ledger snapshot table in the right pane, then click **Approve gather**.
5. Wait for the card to reach `AWAITING_VALIDATE_APPROVAL`. Review the journal entry table, then click **Approve validate**.

**Expected:**
- The new card appears with status `CREATED` within 1 s, then `GATHERING` within 1 s more.
- Within ~30 s the card reaches `AWAITING_GATHER_APPROVAL`. The right pane shows a ledger snapshot table with ≥ 6 rows; each row has a non-empty account code, description, and at least one non-zero amount.
- After the user clicks **Approve gather**, the card transitions to `VALIDATING` within 1 s.
- Within ~30 s more the card reaches `AWAITING_VALIDATE_APPROVAL`. The right pane shows ≥ 2 journal entries; each entry has a non-empty `debitAccount`, `creditAccount`, and `amount`. Accrual entries have a non-empty `reversalDate`.
- After the user clicks **Approve validate**, the card transitions to `REPORTING` within 1 s.
- Within ~30 s more the card reaches `EVALUATED`. The right pane shows a `ClosePackage` with `trialBalance.balanced = true`, and the reconciliation score chip shows 5/5.
- The approval history strip shows two `APPROVED` entries for GATHER and VALIDATE.
- Total elapsed time (excluding human review): ≤ 90 s.

## J2 — Accounting rejects gather phase; run replays and completes

**Preconditions:** Service running. Any model provider (real or mock).

**Steps:**
1. Submit any seeded entity+period.
2. When the card reaches `AWAITING_GATHER_APPROVAL`, click **Reject gather** (optionally add a comment: "Missing Q4 reversal entries").
3. Wait for the card to return to `GATHERING`, then back to `AWAITING_GATHER_APPROVAL`.
4. Click **Approve gather** on the second pass.
5. Complete the remaining validate approval.

**Expected:**
- After **Reject gather**, the card transitions back to `GATHERING` within 1 s. The approval history strip shows one `REJECTED` entry for GATHER.
- The agent reruns the GATHER task. The card reaches `AWAITING_GATHER_APPROVAL` a second time within ~30 s.
- After **Approve gather** on the second pass, the pipeline proceeds normally and reaches `EVALUATED`.
- The approval history strip shows two entries for GATHER (one REJECTED, one APPROVED) and one APPROVED for VALIDATE.

## J3 — Invalid account code flags reconciliation score 1

**Preconditions:** Mock LLM mode. The mock's `write-close-report.json` includes one entry whose first `JournalEntry` references an account code absent from the chart of accounts.

**Steps:**
1. Submit any seeded entity+period four times (the invalid-account-code entry is selected once in every four runs by the mock's `seedFor(closeRunId)` modulo).
2. Complete the approval gates for each run.
3. Watch the live list until a close run's card border highlights red.

**Expected:**
- The flagged close run lands well-formed (the approval gates only check phase order, not account code validity).
- The reconciliation score chip shows **1** and the rationale reads: `"Account-code validity failed: entry 'je-NNN' references account 'XXXX' which does not appear in the chart of accounts."`
- The card's border highlights red. The accounting team knows to inspect this package before filing.
- The other close runs in the batch scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded entity+period.
2. Complete both approval gates.
3. Wait for `EVALUATED`.
4. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the closeRunId.

**Expected:**
- The GATHER task's log entries show only `fetchLedgerLines` and `fetchChartOfAccounts` calls.
- The VALIDATE task's log entries show only `checkDebitCreditBalance` and `validateAccountCodes` calls.
- The REPORT task's log entries show only `formatTrialBalanceSummary` and `writeVarianceCommentary` calls.
- No cross-phase calls appear.
- The approval events for GATHER and VALIDATE appear between their respective task completions and the next task starts, confirming the gate held.

## J5 — Custom entity with no matching ledger file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/ledger/<custom-entity>-<period>.json` exists.

**Steps:**
1. In the App UI, type a custom entity code (e.g. `PHANTOM-CORP`) into the entity field and pick any period.
2. Click **Start close run**.
3. Complete the approval gates when prompted.

**Expected:**
- `GatherTools.fetchLedgerLines` returns an empty list.
- The agent's GATHER task returns a `LedgerSnapshot` with `lines = []`.
- After gather approval, the VALIDATE task returns a `JournalEntrySet` with `entries = []` and narration "(no ledger lines to validate)".
- After validate approval, the REPORT task returns a `ClosePackage` with `trialBalance.balanced = true` (vacuously — zero debits and credits), an empty `journalEntries` list, and a `varianceCommentary` of "(no entries)".
- The reconciliation score chip shows 1 (balance check vacuously passes but no other rules score). The rationale names "no journal entries to evaluate."
- The pipeline completes; nothing crashes; the empty package is honestly empty.

## J6 — Approval history is durable across page reload

**Preconditions:** Service running. Any model provider. At least one close run has reached `EVALUATED` with two approvals.

**Steps:**
1. Complete a close run to `EVALUATED` (J1 steps).
2. Note the `closeRunId` from the UI.
3. Reload the page completely (Ctrl+R).
4. Select the previously completed close run from the live list.

**Expected:**
- The right pane renders the complete close run state: ledger snapshot, journal entry set, close package, reconciliation score chip.
- The approval history strip shows both `APPROVED` entries (GATHER and VALIDATE) with the original approver and timestamp.
- All data matches what was visible before the reload. No loss of approval history across a page reload confirms the entity's event log is the durable source of truth.
