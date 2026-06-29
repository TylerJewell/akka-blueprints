# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a startup profile and watch parallel advisory synthesis

**Preconditions:** service running on `http://localhost:9164/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Fill in company name, sector, stage, and problem statement. Submit.
2. Observe the new advisory session row via SSE.

**Expected:** the session progresses `PLANNING → IN_PROGRESS → SYNTHESISED` within ~90 s. The expanded row shows a `MarketLandscape` (3–5 competitors), a `GtmStrategy` (3–5 channels), a `ContentPlan` (3–5 pillars), a `ProductRoadmap` (3 phases), and an executive summary of 100–150 words. The four specialist outputs arrive close together because the specialists ran in parallel.

## J2 — Worker timeout degrades the session

**Preconditions:** `MarketResearcher` step timeout set to 1 s (test override).

**Steps:**
1. Submit a startup profile.
2. Watch the advisory session.

**Expected:** the `marketStep` times out, the workflow routes to `degradeStep`, and the AdvisorSupervisor synthesises from the three remaining specialist outputs. The session enters `DEGRADED`; the executive summary notes the missing market landscape. No infinite retry.

## J3 — Guardrail blocks a report with fabricated competitor data

**Preconditions:** AdvisorSupervisor returns a report citing a competitor name not present in the supplied `MarketLandscape` (test fixture).

**Steps:**
1. Submit the fixture startup profile.
2. Watch the advisory session.

**Expected:** `guardrailStep` flags the fabricated competitor reference; the workflow calls `block`; the session enters `BLOCKED` with a `failureReason`. The session never appears in the App UI list's finished state.

## J4 — Eval score appears beside a synthesised session

**Preconditions:** at least one `SYNTHESISED` session without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly.
2. Observe the session row.

**Expected:** the session gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `ProfileSimulator` drips a startup profile from `startup-profiles.jsonl` every 60 s; each becomes an advisory session that flows through the full pipeline. The App UI is non-empty on first load.

## J6 — Multiple concurrent sessions make independent progress

**Preconditions:** service running; model provider configured.

**Steps:**
1. Submit three startup profiles in rapid succession without waiting for the first to finish.
2. Watch all three session rows simultaneously.

**Expected:** each session progresses through the state machine independently. No session's specialists block another's. All three sessions reach a terminal state (SYNTHESISED or DEGRADED or BLOCKED) within ~120 s of their respective submissions.
