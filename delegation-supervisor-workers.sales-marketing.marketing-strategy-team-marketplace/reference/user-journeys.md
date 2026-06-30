# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit an objective and watch parallel strategy assembly

**Preconditions:** service running on `http://localhost:9476/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a campaign objective and Submit.
2. Observe the new brief row via SSE.

**Expected:** the brief progresses `PLANNING → IN_PROGRESS → ASSEMBLED` within ~120 s. The expanded row shows a `MarketInsightsBundle` (3–6 insights), an `AudienceProfile` (2–4 segments), a `MessagingGuide` (core message + 3–5 supporting messages), a `ChannelPlan` (3–5 channels), and an assembled executive summary. All four worker payloads arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the brief

**Preconditions:** `MarketResearcher` step timeout set to 1 s (test override).

**Steps:**
1. Submit an objective.
2. Watch the brief.

**Expected:** the `researchStep` times out, the workflow routes to `degradeStep`, and the CampaignDirector assembles from the three remaining worker outputs. The brief enters `DEGRADED`; the executive summary notes the missing market-insights side. No infinite retry.

## J3 — Compliance guardrail blocks a non-compliant brief

**Preconditions:** MessageStrategist returns a coreMessage containing a blocked comparative claim (test fixture).

**Steps:**
1. Submit the fixture objective.
2. Watch the brief.

**Expected:** `guardrailStep` flags the assembled content; the workflow calls `block`; the brief enters `BLOCKED` with a `failureReason` identifying the violation. The brief is never surfaced as a finished result in the App UI list's done state.

## J4 — Eval score appears beside an assembled brief

**Preconditions:** at least one `ASSEMBLED` brief without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly.
2. Observe the brief row.

**Expected:** the brief gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `CampaignSimulator` drips an objective from `campaign-objectives.jsonl` every 60 s; each becomes a brief that flows through the full pipeline. The App UI is non-empty on first load.

## J6 — Four-way parallel fan-out confirmed in logs

**Preconditions:** service running; at least one objective in flight.

**Steps:**
1. Submit an objective.
2. Inspect the service logs during the `IN_PROGRESS` phase.

**Expected:** log entries for `researchStep`, `targetStep`, `messagingStep`, and `channelStep` appear with overlapping timestamps, confirming they ran concurrently. The join step fires only after all four complete (or time out).
