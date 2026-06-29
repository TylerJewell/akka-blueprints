# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a query and watch parallel synthesis

**Preconditions:** service running on `http://localhost:9919/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a financial query (e.g. "Analyse NVDA risk/return for a growth portfolio") and Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `DRAFTING → IN_REVIEW → PUBLISHED` within ~90 s. The expanded row shows a `MarketDataBundle` (3–5 data points), a `PlanningAssessment` (3–5 guidance bullets), a `ReportNarrative` (80–120 word text), and a consolidated executive summary. Market data, planning, and narrative arrive close together because all three workers ran in parallel.

## J2 — Worker timeout degrades the report

**Preconditions:** `MarketResearcher` step timeout set to 1 s (test override).

**Steps:**
1. Submit a financial query.
2. Watch the report row.

**Expected:** the `gatherMarketStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the PortfolioPlanner and ReportDrafter outputs. The report enters `DEGRADED`; the executive summary notes the missing market-data side. No infinite retry.

## J3 — Guardrail blocks a non-compliant report

**Preconditions:** Coordinator returns a report with a direct securities recommendation ("buy NVDA now" or "guaranteed upside") that the sanitizer cannot remediate (test fixture).

**Steps:**
1. Submit the fixture query.
2. Watch the report row.

**Expected:** `guardrailStep` flags the consolidated content; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason` describing the violation. The report is never surfaced as a published result in the App UI list.

## J4 — Finance-sector sanitizer replaces regulated phrases

**Preconditions:** Coordinator returns a report containing "risk-free return" (test fixture).

**Steps:**
1. Submit the fixture query.
2. Inspect the published report's `consolidated.sanitizerVerdict`.

**Expected:** the report reaches `PUBLISHED`; `sanitizerVerdict` records the replacement made (e.g. "Replaced 'risk-free return' with 'lower-risk return'"). The final text does not contain the original phrase. Delivery was not blocked.

## J5 — Eval score appears beside a published report

**Preconditions:** at least one `PUBLISHED` report without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it.
2. Observe the report row.

**Expected:** the report gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score badge. Delivery of the report was never delayed by the eval (non-blocking).

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `QuerySimulator` drips a query from `financial-queries.jsonl` every 60 s; each becomes a report that flows through the full pipeline. The App UI is non-empty on first load.
