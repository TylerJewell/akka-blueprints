# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit an incident and watch parallel triage

**Preconditions:** service running on `http://localhost:9464/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Fill in an asset ID (e.g., `prod-api-gateway-01`), asset type (`api-gateway`), signal description, reporter, and initial severity `HIGH`. Click Submit.
2. Observe the new incident row via SSE.

**Expected:** the incident progresses `RECEIVED → TRIAGING → AWAITING_APPROVAL` within ~60 s. Vulnerabilities and threat context arrive close together because the workers ran in parallel. The row shows Approve and Reject buttons once the status is `AWAITING_APPROVAL`.

## J2 — Approve the mitigation

**Preconditions:** at least one incident in `AWAITING_APPROVAL`.

**Steps:**
1. Click the Approve button on the incident row.
2. Enter an officer ID (e.g., `officer-jane-smith`) and an optional reason.
3. Submit.

**Expected:** the incident moves to `MITIGATED` via SSE within a few seconds. The expanded row shows the approval decision (officer ID, reason, timestamp). No mitigation was attempted before the approval was recorded.

## J3 — Reject the mitigation

**Preconditions:** at least one incident in `AWAITING_APPROVAL`.

**Steps:**
1. Click the Reject button on the incident row.
2. Enter an officer ID and a required reason.
3. Submit.

**Expected:** the incident moves to `REJECTED` via SSE. The expanded row shows the rejection reason. The mitigation plan is not executed.

## J4 — Worker timeout degrades the incident

**Preconditions:** `VulnerabilityScanner` step timeout set to 1 s (test override).

**Steps:**
1. Submit an incident.
2. Watch the incident.

**Expected:** the `scanStep` times out, the workflow routes to `degradeStep`, and the SecurityCoordinator triages from the ThreatContextBundle alone. The incident enters `DEGRADED`; the triage summary notes the missing vulnerability scan. No infinite retry; no approval gate is reached.

## J5 — Eval score appears beside a mitigated incident

**Preconditions:** at least one `MITIGATED` incident without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the incident row.

**Expected:** the incident gains an `evalScore` (1–5) and an `evalRationale`; the App UI row displays the score. Delivery was never blocked by the eval — the `MITIGATED` state was reached before the eval ran (non-blocking).

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `IncidentSimulator` drips an incident signal from `incident-signals.jsonl` every 90 s; each becomes an incident that flows through the triage pipeline and parks at `AWAITING_APPROVAL`. The App UI is non-empty on first load.
