# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a valid property deal and watch parallel evaluation

**Preconditions:** service running on `http://localhost:9581/`; a model provider configured (real or mock); property type not on the exclusion list.

**Steps:**
1. Open the App UI tab. Enter an address, select "multifamily" as property type, enter an asking price, and Submit.
2. Observe the new deal row via SSE.

**Expected:** the deal progresses `SCOPING → EVALUATING → RECOMMENDED` within ~60 s. The expanded row shows a `MarketReport` (3–6 comparable sales, neighborhood trend, estimated market value), a `FinancialModel` (cap rate, NOI, cash-on-cash return, DSCR), and a recommendation summary. The market report and financial model arrive close together because the specialists ran in parallel.

## J2 — Sector sanitizer rejects an excluded property type

**Preconditions:** service running; the property type field is set to `"cannabis_facility"`.

**Steps:**
1. Submit a deal with property type `cannabis_facility`.
2. Watch the deal row appear via SSE.

**Expected:** the deal enters `REJECTED` immediately — no agent task is scheduled. The `failureReason` field explains the exclusion. No `MarketReport` or `FinancialModel` is attached. The deal never transitions through `EVALUATING`.

## J3 — Specialist timeout degrades the deal

**Preconditions:** `MarketSpecialist` step timeout set to 1 s (test override).

**Steps:**
1. Submit a valid property deal.
2. Watch the deal.

**Expected:** the `marketStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the FinancialModel alone. The deal enters `DEGRADED`; the summary notes the missing market data. No infinite retry.

## J4 — Output guardrail blocks a flawed recommendation

**Preconditions:** DealCoordinator returns a recommendation with a fabricated cap rate (test fixture).

**Steps:**
1. Submit the fixture deal.
2. Watch the deal.

**Expected:** `guardrailStep` flags the synthesised content; the workflow calls `block`; the deal enters `BLOCKED` with a `failureReason`. The recommendation is never surfaced as a finished result in the App UI list's done state.

## J5 — Eval score appears beside a recommended deal

**Preconditions:** at least one `RECOMMENDED` deal without an `evalScore`.

**Steps:**
1. Wait for `RecommendationEvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the deal row.

**Expected:** the deal gains an `evalScore` (1–5) and an `evalRationale`; the App UI row displays the score. Delivery of the recommendation was never blocked by the eval (non-blocking).

## J6 — Background load from the deal simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `DealSimulator` drips a scenario from `sample-deals.jsonl` every 60 s; each becomes a deal that flows through the full pipeline (sanitizer → scope → parallel specialists → synthesise). The App UI is non-empty on first load.
