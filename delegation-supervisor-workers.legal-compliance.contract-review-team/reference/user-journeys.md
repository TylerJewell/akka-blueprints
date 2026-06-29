# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a contract and watch parallel review

**Preconditions:** service running on `http://localhost:9533/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a `contractRef` (e.g., `NDA-2026-Test`), paste contract text, enter your name in `submittedBy`, and click Submit.
2. Observe the new review row via SSE.

**Expected:** the review progresses `QUEUED → IN_REVIEW → AWAITING_APPROVAL` within ~90 s. The expanded row shows a `ClauseSummary` (3–8 clauses), a `RiskReport` (at least one risk flag), and a `RedlineSet` (suggestions for HIGH/CRITICAL clauses). Clause summary, risk report, and redlines arrive close together because the workers ran in parallel. Approve and Reject buttons appear on the row.

## J2 — Worker timeout degrades the review

**Preconditions:** `Redliner` step timeout set to 1 s (test override in application.conf or WorkflowSettings).

**Steps:**
1. Submit a contract.
2. Watch the review row.

**Expected:** the `redlineStep` times out, the workflow routes to `degradeStep`, and the supervisor consolidates from the ClauseSummary and RiskReport alone. The review enters `DEGRADED`; the expanded row notes that redlines are unavailable. No infinite retry.

## J3 — Sanitizer blocks a prohibited contract

**Preconditions:** submit a contract text that includes a clause the legal-sector sanitizer's ruleset classifies as prohibited (e.g., a clause explicitly mandating discriminatory treatment of parties based on a protected characteristic).

**Steps:**
1. Enter the prohibited contract text and click Submit.
2. Watch the review row.

**Expected:** `sanitizeStep` fires before any agent runs, the workflow calls `rejectBySanitizer`, and the review enters `REJECTED` with a `failureReason`. The status pill on the row shows `REJECTED` and no clause summary, risk report, or redlines are present.

## J4 — Guardrail blocks a problematic consolidated package

**Preconditions:** use a test fixture where `ReviewSupervisor` returns a `ReviewPackage` containing a redline suggestion that the guardrail's ruleset classifies as legally impermissible (e.g., a proposed clause that waives a mandatory consumer protection right).

**Steps:**
1. Submit the fixture contract.
2. Watch the review row.

**Expected:** `guardrailStep` flags the package; the workflow calls `block`; the review enters `BLOCKED` with a `failureReason`. The row shows `BLOCKED` and no Approve/Reject buttons appear.

## J5 — Lawyer approves a review

**Preconditions:** at least one review is in `AWAITING_APPROVAL`.

**Steps:**
1. Find the AWAITING_APPROVAL row in the App UI.
2. Click the Approve button. Enter a lawyer note when prompted (e.g., "Redlines accepted; IP clause scope is acceptable."). Confirm.

**Expected:** the review transitions to `APPROVED`. The `lawyerNote` is visible in the expanded row. The Approve and Reject buttons disappear. The `finishedAt` timestamp is set.

## J6 — Eval score appears beside an approved review

**Preconditions:** at least one `APPROVED` review without an `evalScore`.

**Steps:**
1. Wait for `ReviewSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the approved review row.

**Expected:** the review gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Review delivery was never blocked by the eval (non-blocking).
