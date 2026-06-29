# User journeys — churn-monitor

## J1 — Account snapshot arrives and is scored HIGH

**Preconditions:** Service running on declared port; valid model-provider API key set; `AccountPoller` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 60 s for the first simulated account snapshot.

**Expected:**
- Account appears with status RECEIVED, then within 1 s transitions to SANITIZED. The sanitized account name (`[REDACTED-NAME]`) is visible; the raw name is not displayed.
- Within 2 s the status transitions to SCORED with a HIGH risk level chip and probability value.
- Within 30 s the status reaches AWAITING_ACTION with a retention plan visible in the detail pane, listing 2–4 ranked actions.

## J2 — Manager marks a HIGH-risk account as actioned

**Preconditions:** An account in AWAITING_ACTION.

**Steps:**
1. Select the account in the risk board.
2. Review the retention plan in the detail pane.
3. Click **Mark Actioned**.

**Expected:**
- Status transitions to ACTIONED immediately. The detail pane shows `decidedBy` set from the UI's session identifier.
- An `ActionRecorded` event is written to the entity.
- The card moves to its terminal styling (muted, no action buttons).

## J3 — Manager dismisses a HIGH-risk account

**Preconditions:** An account in AWAITING_ACTION.

**Steps:**
1. Click **Dismiss**. A reason textarea opens.
2. Enter a reason (e.g., "Account confirmed renewal by phone"); click Confirm.

**Expected:**
- Status transitions to DISMISSED. `decision.reason` is populated and visible on the card.
- The account moves to its terminal styling.

## J4 — LOW and MEDIUM risk accounts auto-close

**Preconditions:** Service running; at least one LOW and one MEDIUM risk account seeded in `account-snapshots.jsonl`.

**Steps:**
1. Wait for the LOW and MEDIUM accounts to be processed by the poller.

**Expected:**
- LOW risk account transitions through RECEIVED → SANITIZED → SCORED → LOW_RISK_CLOSED without appearing in the AWAITING_ACTION column.
- MEDIUM risk account transitions through RECEIVED → SANITIZED → SCORED → MEDIUM_RISK_CLOSED without requiring manager action.
- Neither account shows action buttons in the UI.

## J5 — Drift and fairness eval runs and surfaces results

**Preconditions:** At least 5 SCORED accounts exist. `DriftFairnessEvalRunner` schedule reduced for test (`EVAL_RUNNER_SECONDS=60`).

**Steps:**
1. Wait for accounts to be scored.
2. Wait 60 s for the eval runner to fire.

**Expected:**
- At least one account's detail pane shows an `EvalCompleted` chip with a `driftVerdict` and `fairnessVerdict`.
- `GET /api/churn/{accountId}` includes a populated `latestEval` field.
- The Eval Matrix tab on the UI shows the two eval rows (S1, E1) with the latest run metadata visible.

## J6 — PII does not reach the LLM (audit check)

**Preconditions:** Service running with debug logging enabled (`LOG_LEVEL=DEBUG`).

**Steps:**
1. Trigger a custom snapshot via `POST /api/churn` (or via the poller's next tick) with `accountName="Alice Nguyen"`, `contactEmail="alice@example.com"`.
2. Inspect the service log for the LLM call payload sent to `ChurnScoringAgent`.

**Expected:**
- The log shows `[REDACTED-NAME]` and no email address in the LLM input — never `Alice Nguyen` or `alice@example.com`.
- The entity's `snapshot.accountName` (the audit log) still holds `"Alice Nguyen"`.
- The entity's `sanitized.redactedAccountName` holds `"[REDACTED-NAME]"`.
- `sanitized.piiCategoriesFound` includes `"account-name"` and `"email"`.
