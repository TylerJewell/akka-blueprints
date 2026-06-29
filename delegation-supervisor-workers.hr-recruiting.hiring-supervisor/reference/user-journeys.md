# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit an application and watch parallel evaluation

**Preconditions:** service running on `http://localhost:9652/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a candidateId, roleId, resume text, and availability window. Submit.
2. Observe the new application row via SSE.

**Expected:** the application progresses `RECEIVED → UNDER_REVIEW → RECOMMENDED` (or `REJECTED`) within ~90 s. The expanded row shows a `ScreeningReport` (qualificationScore + strength highlights + gap notes), a `SchedulingProposal` (2–3 slots), and a hiring recommendation with rationale. The screening report and scheduling proposal arrive close together because the two agents ran in parallel.

## J2 — Worker timeout degrades the application

**Preconditions:** `ScreeningAgent` step timeout set to 1 s (test override).

**Steps:**
1. Submit an application.
2. Watch the application row.

**Expected:** the `screenStep` times out, the workflow routes to `degradeStep`, and `HiringSupervisor` consolidates from the `SchedulingProposal` alone. The application enters `DEGRADED`; the recommendation rationale notes the missing screening side. No infinite retry.

## J3 — Guardrail holds an application before a delegate call

**Preconditions:** delegation policy configured to block a specific roleId (test fixture).

**Steps:**
1. Submit an application with the blocked roleId.
2. Watch the application row.

**Expected:** the before-invocation guardrail fires in `planStep` before any sub-agent call is issued; the workflow calls `policyHold`; the application enters `POLICY_HOLD` with a structured `failureReason`. No `ScreeningAgent` or `SchedulingAgent` call is made.

## J4 — Eval score appears beside a recommended application

**Preconditions:** at least one `RECOMMENDED` application without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the application row.

**Expected:** the application gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery of the recommendation was never blocked by the eval (non-blocking).

## J5 — Sanitizer removes protected-attribute fields before agents are called

**Preconditions:** service running; a candidate application submitted that includes a field value suggesting age or gender in the raw `resumeText` payload.

**Steps:**
1. Submit the application.
2. After the workflow completes, inspect the `ApplicationEntity` state via `GET /api/hiring/{id}`.

**Expected:** the stored `CandidateApplication` contains no explicit protected-attribute fields; the `ScreeningReport` and `SchedulingProposal` do not reference gender, age, disability status, or nationality. The `sanitized` flag on `SanitizedApplication` was `true` for every agent call.

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `CandidatePipelineSimulator` drips an application from `candidate-applications.jsonl` every 90 s; each becomes an application that flows through the full pipeline. The App UI is non-empty on first load.
