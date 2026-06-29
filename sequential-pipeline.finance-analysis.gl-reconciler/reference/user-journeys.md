# User journeys — gl-reconciler

## J1 — Submit an account set and post a journal entry

**Preconditions:** Service running on declared port (`http://localhost:9371/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded account set `fund-a-q2-close` has a matching `src/main/resources/sample-data/accounts/fund-a-q2-close.json` file. No variance in that file exceeds the material threshold.

**Steps:**
1. Open `http://localhost:9371/` → App UI tab.
2. From the **Pick a seeded account set** dropdown, pick `Fund A — Q2 close`.
3. Click **Run reconciliation**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `FETCHING` within 1 s more.
- Within ~20 s the card reaches `FETCHED`. The right pane shows the Ledger entries table with ≥ 4 rows; each row has a non-empty account ID, description, and debit or credit amount.
- Within ~20 s more the card reaches `RECONCILED`. The right pane shows a variance table with one row per account and a NAV line. All `isMaterial` flags are false; the card does NOT show an amber dot.
- Within ~20 s more the card reaches `DRAFT_VALIDATED`, then `POSTED` within 1 s. The right pane shows a balanced journal entry: `totalDebits == totalCredits`, balance-check chip shows ✓.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Posting guardrail rejects an unbalanced journal draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-journal.json` includes one entry whose `totalDebits != totalCredits` — selected on the FIRST iteration of every 4th run (modulo seed).

**Steps:**
1. Submit any seeded account set four times in a row (J1 steps × 4).
2. Watch the fourth submission's lifecycle in the network panel of the browser dev tools (`/api/reconciliations/sse`).

**Expected:**
- On the fourth run's `draftStep`, the agent's first iteration returns an unbalanced `JournalEntry`. `PostingGuardrail` rejects it; a `ValidationFailed{findings: ["balance-rule: totalDebits 500.00 != totalCredits 450.00"]}` event lands on the entity.
- The unbalanced draft NEVER reaches `JournalDrafted` — there is no log line recording it on the entity.
- The agent's second iteration returns a balanced draft. The card eventually reaches `POSTED` as in J1.
- The card in the App UI shows the validation-failure strip with the one rejected draft's finding. The strip disappears once the run posts successfully; the finding is still in the entity log.

## J3 — Material variance triggers HITL escalation

**Preconditions:** Service running. Any model provider. The seeded account set `Multi-currency sweep — June` includes account 2300 with a delta of EUR 2300.00, which exceeds the 1 % material threshold.

**Steps:**
1. From the dropdown pick `Multi-currency sweep — June`.
2. Click **Run reconciliation**.
3. When the card reaches `PENDING_APPROVAL`, observe the amber dot and the escalation strip in the right pane.
4. Click **Approve** (enter reviewer name `j.smith@example.com`).

**Expected:**
- The card transitions through `FETCHING → FETCHED → RECONCILING → RECONCILED` and then pauses at `PENDING_APPROVAL`.
- The right pane shows the escalation strip with the reason: _"Material variance in account 2300: delta EUR 2300.00 exceeds 1% threshold (EUR 500.00)."_
- The `Draft journal` phase panel is NOT visible yet — the workflow has not proceeded to `draftStep`.
- After clicking **Approve**, the card transitions through `DRAFTING → DRAFT_VALIDATED → POSTED` within ~30 s.
- The right pane's escalation strip now shows `reviewedBy: j.smith@example.com`, `decision: APPROVED`, and `decidedAt`.

## J4 — Controller rejects the escalation

**Preconditions:** Same as J3.

**Steps:**
1. Run the `Multi-currency sweep — June` account set again.
2. When the card reaches `PENDING_APPROVAL`, click **Reject** and enter reason `"Variance in account 2300 requires GL team investigation."`.

**Expected:**
- The card transitions from `PENDING_APPROVAL` to `REJECTED` within 1 s.
- The right pane shows the escalation strip with `decision: REJECTED` and the entered reason.
- No `JournalDrafted`, `ValidationPassed`, or `JournalPosted` events appear in the entity log — the workflow terminated before `draftStep`.
- The card stays visible in the live list in `REJECTED` state with the partial data (snapshot and reconciliation populated, journal null).

## J5 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded account set.
2. Wait for `POSTED`.
3. Inspect the service log for tool-call lines. Filter to entries tagged with the runId.

**Expected:**
- The FETCH task's log entries show only `fetchLedgerEntries` and `fetchAccountBalances` calls.
- The RECONCILE task's log entries show only `computeVariances` and `calculateNAV` calls.
- The DRAFT task's log entries show only `formatJournalLine` and `sumJournalLines` calls.
- No cross-phase calls appear.
- The order of tasks in the log is FETCH → RECONCILE → DRAFT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J6 — Account set with no matching sample file (custom input)

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/accounts/<custom-id>.json` exists for the user's input.

**Steps:**
1. In the App UI, type a custom account set identifier (e.g. `test-fund-xyz`) into the account set field.
2. Click **Run reconciliation**.

**Expected:**
- `FetchTools.fetchLedgerEntries` returns an empty list; `fetchAccountBalances` returns an empty list.
- The agent's FETCH task returns a `LedgerSnapshot` with `entries = []` and `expectedBalances = []`.
- The RECONCILE task returns a `ReconciliationResult` with `variances = []`, `allAccountsReconciled = true`, `hasMaterialVariance = false`.
- No escalation is raised (no material variances).
- The DRAFT task returns a `JournalEntry` with `lines = []`, `totalDebits = 0`, `totalCredits = 0`.
- `PostingGuardrail` accepts the empty journal (balance rule: 0 == 0; account-code rule: no lines to check).
- The card reaches `POSTED`. Nothing crashes; the empty journal is honestly empty.
