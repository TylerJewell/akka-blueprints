# User journeys — `recruitment-team`

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit an application and watch a recommendation appear

- **Preconditions:** service running on `http://localhost:9654/`; a model provider configured (real or mock).
- **Steps:** POST `/api/applications` with a `roleId` and `resume`. Subscribe to `/api/candidates/sse`.
- **Expected:** the candidate appears in `SANITIZED`, then `MATCHED`, `SCREENED`, and lands in `AWAITING_DECISION` within ~30 s with a non-empty `supervisorSummary`, a `matchScore`, and an `evalScore`. `sanitizedResume` shows redaction placeholders.

## J2 — Approve a candidate

- **Preconditions:** a candidate in `AWAITING_DECISION`.
- **Steps:** click Approve in the App UI (POST `/api/candidates/{id}/approve`).
- **Expected:** status becomes `APPROVED`; `decidedBy` and `decidedAt` are set; Approve/Reject buttons disappear.

## J3 — Reject a candidate with a reason

- **Preconditions:** a candidate in `AWAITING_DECISION`.
- **Steps:** click Reject, enter a reason (POST `/api/candidates/{id}/reject`).
- **Expected:** status becomes `REJECTED`; `decisionReason` renders in the row.

## J4 — A stale decision escalates

- **Preconditions:** a candidate left in `AWAITING_DECISION` past the 2-minute window with no human action.
- **Steps:** wait for `StuckDecisionMonitor` to run.
- **Expected:** status becomes `ESCALATED`; `escalatedAt` is set; Approve/Reject buttons disappear.

## J5 — Fairness drift is flagged

- **Preconditions:** `ApplicationSimulator` seeding applications so several candidates reach decided states.
- **Steps:** let `FairnessDriftMonitor` run across recent decisions.
- **Expected:** when recent outcomes skew past the threshold, at least one candidate has `driftFlagged = true` and a drift badge appears in the UI.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix shows four controls (S1, E1, P1, H1) in `matrix-card` style with colored mechanism pills; Risk Survey shows pre-filled answers with deployer placeholders muted.
