# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a candidate site and watch parallel scoring

**Preconditions:** service running on `http://localhost:9747/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter an address, city, and region, then Submit.
2. Observe the new site row via SSE.

**Expected:** the evaluation progresses `SCORING → IN_PROGRESS → RECOMMENDED` (or `NOT_RECOMMENDED`) within ~60 s. The expanded row shows a `MarketAssessment` (trafficScore, competitorDensity, tradeAreaClassification, 3–5 keyInsights) and a `DemographicAssessment` (populationScore, incomeAlignmentScore, consumerProfile, 3–5 keyInsights), plus a synthesised recommendation summary and composite score. Both assessments arrive close together because the analysts ran in parallel.

## J2 — Analyst timeout degrades the evaluation

**Preconditions:** `MarketAnalyst` step timeout set to 1 s (test override).

**Steps:**
1. Submit a candidate site.
2. Watch the evaluation row.

**Expected:** the `assessMarketStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the Demographics Analyst output alone. The evaluation enters `DEGRADED`; the summary notes the missing market dimension and flags lower confidence. No infinite retry.

## J3 — Guardrail blocks a flawed recommendation

**Preconditions:** LocationCoordinator returns a recommendation with an out-of-range score (test fixture — score > 1.0).

**Steps:**
1. Submit the fixture candidate site.
2. Watch the evaluation row.

**Expected:** `guardrailStep` detects the out-of-range score; the workflow calls `block`; the evaluation enters `BLOCKED` with a `failureReason` naming which check failed. The evaluation is never surfaced as a finished recommendation in the App UI.

## J4 — Eval score appears beside a finished recommendation

**Preconditions:** at least one `RECOMMENDED` or `NOT_RECOMMENDED` evaluation without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the site row in the App UI.

**Expected:** the evaluation gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score badge. Delivery of the recommendation was never blocked by the eval process (non-blocking enforcement).

## J5 — Background load from the site simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `SiteSimulator` drips a candidate site from `candidate-sites.jsonl` every 60 s; each becomes an evaluation that flows through the full pipeline. The App UI is non-empty on first load.
