# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a non-production change and watch parallel assessment

**Preconditions:** service running on `http://localhost:9794/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Select environment "staging", change type "deploy", enter a description, and Submit.
2. Observe the new change-request row via SSE.

**Expected:** the change request progresses `PLANNING → IN_PROGRESS → APPROVED` within ~90 s. The expanded row shows an `InfraAssessment`, a `DeployAssessment`, an `ObsAssessment`, and a consolidated readiness report summary. All three assessments arrive close together because the specialists ran in parallel.

## J2 — Submit a production change and approve it

**Preconditions:** service running; model provider configured.

**Steps:**
1. Select environment "production", change type "deploy", and Submit.
2. Observe the change-request row move to `AWAITING_APPROVAL`.
3. Expand the row; click **Approve**; enter your name in the prompt.

**Expected:** the change moves to `APPROVED`. The `approvedBy` field is populated with the entered name. The workflow resumes and emits `ChangeApproved`.

## J3 — Guardrail blocks a destructive tool call

**Preconditions:** a specialist agent's tool set includes a blocklisted call (test fixture, e.g., `forceDeletePod`).

**Steps:**
1. Submit a change request that triggers the blocklisted tool call (use the designated fixture topic from `change-requests.jsonl`).
2. Watch the change-request row.

**Expected:** the before-tool-call guardrail fires before the tool executes; the workflow calls `ChangeRequestEntity.block`; the change enters `BLOCKED` with a `failureReason`. The guardrail fires before any tool-side effect occurs.

## J4 — Operator halt stops in-flight workflows

**Preconditions:** at least one change request in `IN_PROGRESS` or `AWAITING_APPROVAL`.

**Steps:**
1. Click **Issue Halt** in the UI header; enter a reason in the prompt.
2. Observe the in-flight change-request rows.

**Expected:** `HaltMonitor` detects the active signal within 30 s; each in-flight workflow receives a halt command and transitions to `HALTED`. Rows in `AWAITING_APPROVAL` also halt — the approval buttons disappear. No further pipeline events are processed while the halt signal is active.

## J5 — Specialist timeout degrades a change request

**Preconditions:** `DeployAgent` step timeout set to 1 s (test override).

**Steps:**
1. Submit any change request.
2. Watch the row.

**Expected:** the `deployStep` times out; the workflow routes to `degradeStep`; the Coordinator consolidates from `InfraAssessment` and `ObsAssessment` alone. The change enters `DEGRADED`; the readiness report summary notes the missing deploy assessment. No infinite retry occurs.

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `PipelineSimulator` drips a change request from `change-requests.jsonl` every 90 s; each flows through the full pipeline (or approval gate for production requests). The App UI is non-empty on first load.
