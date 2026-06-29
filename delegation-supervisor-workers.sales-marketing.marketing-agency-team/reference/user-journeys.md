# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a campaign brief and watch parallel planning

**Preconditions:** service running on `http://localhost:9404/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a campaign name and objective, then Submit.
2. Observe the new plan row via SSE.

**Expected:** the plan progresses `PLANNING → IN_PROGRESS → SYNTHESISED` within ~60 s. The expanded row shows a `LaunchBrief` (3–6 checklist items with priorities), a `StrategyFramework` (3–5 messaging pillars, 2–4 target audiences), and a synthesised executive summary. The launch brief and strategy framework arrive close together because both specialists ran in parallel.

## J2 — Specialist timeout degrades the plan

**Preconditions:** `WebsiteLauncher` step timeout set to 1 s (test override).

**Steps:**
1. Submit a campaign brief.
2. Watch the plan.

**Expected:** the `launchStep` times out, the workflow routes to `degradeStep`, and the CampaignDirector synthesises from the StrategyAdvisor output alone. The plan enters `DEGRADED`; the executive summary notes the missing launch brief. No infinite retry.

## J3 — Brand guardrail blocks a non-compliant plan

**Preconditions:** CampaignDirector returns a plan with an unsubstantiated superlative claim (test fixture).

**Steps:**
1. Submit the fixture campaign brief.
2. Watch the plan.

**Expected:** `guardrailStep` flags the synthesised content; the workflow calls `block`; the plan enters `BLOCKED` with a `failureReason`. The plan is not surfaced as a finished result in the App UI's done state.

## J4 — Eval score appears beside a synthesised plan

**Preconditions:** at least one `SYNTHESISED` plan without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the plan row.

**Expected:** the plan gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Plan delivery was not blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running for at least 60 seconds.

**Expected:** `RequestSimulator` drips a scenario from `campaign-scenarios.jsonl` every 60 s; each becomes a plan that flows through the full pipeline. The App UI is non-empty on first load.
