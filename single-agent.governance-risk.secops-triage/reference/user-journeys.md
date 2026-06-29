# User journeys — secops-triage

## J1 — Ingest a critical finding and receive a triage verdict

**Preconditions:** Service running on declared port (`http://localhost:9190/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9190/` → App UI tab.
2. From the **Load seeded finding** dropdown, pick `Container escape (CVSS 9.8)`.
3. Click **Ingest finding**.

**Expected:**
- The new card appears in the live list with status `INGESTED` within 1 s.
- The card transitions to `ENRICHED` within 1 s. The right-pane detail shows asset criticality `TIER1`, `internetFacing: true`, and the threat-intel summary indicating an exploit in the wild.
- Within 30 s the card reaches `VERDICT_RECORDED`. The right pane shows: priority badge `CRITICAL_IMMEDIATE`, a 2–4 sentence risk rationale referencing CVSS 9.8 and exploit availability, and the recommended action beginning with an actionable verb.
- Within 1 s of `VERDICT_RECORDED` (because `requiresApproval = true`), the card transitions to `PENDING_APPROVAL` and the approve/reject panel appears in the right pane.

## J2 — Analyst approves a remediation

**Preconditions:** J1 completed; finding is in `PENDING_APPROVAL`.

**Steps:**
1. In the right pane, enter `analyst-jsmith` in the Analyst ID field and a reason in the Reason field.
2. Click **Approve**.

**Expected:**
- `POST /api/findings/{id}/approve` returns 204.
- The finding transitions to `REMEDIATED` via SSE within 1 s.
- The approve/reject buttons disappear. The approval result section shows the analyst id, `GRANTED` badge, reason, and timestamp.
- The card in the live list updates its status pill to `REMEDIATED` (green).

## J3 — Before-tool-call guardrail intercepts a remediation action

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock agent's task response includes a `patch-deploy` tool call.

**Steps:**
1. Ingest the container-escape seeded finding.
2. Monitor the SSE stream at `/api/findings/sse` in the browser network panel.

**Expected:**
- The finding transitions to `TRIAGING` as expected.
- Before the triage verdict lands, the SSE stream emits a `PENDING_APPROVAL` transition — indicating the guardrail intercepted the remediation tool call during the agent's execution.
- The verdict's `requiresApproval` field is `true`. The `ApprovalEntity` record exists (observable via `GET /api/findings/{id}`).
- No remediation action executed on any infrastructure — the finding remains in `PENDING_APPROVAL` until the analyst acts.
- The service log shows one `guardrail.block` line with the intercepted tool-call name.

## J4 — Periodic drift evaluator raises a drift alert

**Preconditions:** Mock LLM mode. The mock returns `LOW_ACCEPTED` verdicts for the last 30+ findings to push the `LOW_ACCEPTED` percentage above 60%.

**Steps:**
1. Ingest 30 findings in rapid succession using the misconfiguration seeded example (which the mock maps to `LOW_ACCEPTED`).
2. Wait for `DriftCheckWorkflow` to trigger (up to 6 hours in real time; the drift check can be triggered immediately in tests by calling the internal drift-check endpoint).

**Expected:**
- The drift evaluator computes that `LOW_ACCEPTED` exceeds 60% of recent findings.
- A `drift-alert` SSE event is emitted.
- The App UI tab displays a yellow drift-alert banner with the description and baseline vs. observed percentages.
- No individual finding is blocked or modified; the alert is advisory.

## J5 — Medium-severity finding skips the approval gate

**Preconditions:** Service running; any model provider.

**Steps:**
1. From the **Load seeded finding** dropdown, pick `Misconfiguration (CVSS 5.3)`.
2. Click **Ingest finding**.
3. Wait for `VERDICT_RECORDED`.

**Expected:**
- The verdict priority is `MEDIUM_MONITORED`. `requiresApproval` is `false`.
- The finding transitions directly to `MONITORED` without entering `PENDING_APPROVAL`.
- No approve/reject panel appears in the UI.
- The card's status pill turns teal (`MONITORED`).

## J6 — Analyst rejects a remediation action

**Preconditions:** J1 completed; finding is in `PENDING_APPROVAL`.

**Steps:**
1. In the right pane, enter `analyst-jsmith` in the Analyst ID field and `Action targets wrong host — re-submit with correct asset ID.` in the Reason field.
2. Click **Reject**.

**Expected:**
- `POST /api/findings/{id}/reject` returns 204.
- The finding transitions to `REMEDIATION_REJECTED` via SSE within 1 s.
- The approve/reject buttons disappear. The approval result section shows the analyst id, `REJECTED` badge, and the reason string.
- The card in the live list updates its status pill to `REMEDIATION_REJECTED` (red).
- The raw finding and triage verdict remain visible in the right pane for reference when re-submitting.
