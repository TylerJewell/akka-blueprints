# User journeys — aws-audit-monitor

## J1 — Finding arrives and reaches PENDING_REVIEW

**Preconditions:** Service running on port 9246; valid model-provider API key set; `AuditScanPoller` enabled.

**Steps:**
1. Open `http://localhost:9246/` → App UI tab.
2. Wait up to 60 s for the first simulated finding.

**Expected:**
- Finding card appears with status DETECTED, then within 1 s transitions to NORMALIZED. The normalized detail is visible; the raw ARN with the account ID segment is masked.
- Within 20 s the analysis chip appears with a severity badge (CRITICAL/HIGH/MEDIUM/LOW/INFO) and the control reference.
- Within 90 s the finding is COMPILED and its associated report card enters PENDING_REVIEW. The Publish and Dismiss controls become active on the report panel.

## J2 — Risk officer publishes a report

**Preconditions:** A report in PENDING_REVIEW.

**Steps:**
1. Select the report in the reports panel.
2. Review the executive summary and the finding list.
3. Click Publish. The reviewer identifier is set from the UI's session id.

**Expected:**
- Report status transitions to PUBLISHED immediately. All linked findings transition to PUBLISHED.
- `ReportPublished` and `FindingPublished` events are written to their respective entities; no actual outbound notification leaves the process (the notify tool is a no-op stub).
- Report and finding cards move to their terminal styling.

## J3 — Risk officer dismisses a finding

**Preconditions:** A finding in PENDING_REVIEW.

**Steps:**
1. Select the finding.
2. Click Dismiss. A reason textarea opens.
3. Type a reason; click Confirm.

**Expected:**
- Finding status transitions to DISMISSED. `decision.published = false`, `decision.reason` populated.
- The reason is visible on the finding card.
- If all findings in the associated report are dismissed, the report also transitions to DISMISSED.

## J4 — Accuracy eval score appears

**Preconditions:** At least one PUBLISHED finding exists with no `evalScore`. The `AccuracyEvalRunner` schedule reduced for the test (`EVAL_RUNNER_SECONDS=120`).

**Steps:**
1. Publish a report.
2. Wait 120 s.

**Expected:**
- The finding card shows an eval score chip (1–5) and the eval rationale is visible in the detail pane.
- `GET /api/audit/findings/{id}` includes `evalScore` populated.

## J5 — Account identifiers do not reach the LLM

**Preconditions:** Service running with debug logging enabled (`LOG_LEVEL=DEBUG`).

**Steps:**
1. Observe the service log during a scan tick.
2. Locate the LLM call payload in the log.

**Expected:**
- The log shows `FindingAnalystAgent`'s input contains the masked ARN form (e.g., `arn:aws:s3:::my-bucket`) — never the raw account-ID segment (e.g., `arn:aws:s3:::123456789012:my-bucket`).
- The `FindingQueue` entity's `FindingDetected.raw` still has the original un-masked ARN; the `AuditFindingEntity`'s `normalized.resourceArn` has the masked form.

## J6 — Multiple findings compile into one report

**Preconditions:** At least 3 scan ticks have completed, producing findings in the same batch window.

**Steps:**
1. Open the Reports panel in the App UI.

**Expected:**
- A single report card lists all findings from the batch, ordered by severity (CRITICAL first).
- The executive summary correctly counts the total and breaks down by severity.
- Clicking Publish on the report transitions all linked findings to PUBLISHED simultaneously.
