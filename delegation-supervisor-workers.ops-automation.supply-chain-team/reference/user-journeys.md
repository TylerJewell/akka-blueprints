# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a demand signal and watch parallel optimization

**Preconditions:** service running on `http://localhost:9488/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a SKU (e.g., `SKU-ELEC-9821`) and a target quantity (e.g., `500`), then Submit.
2. Observe the new order row via SSE.

**Expected:** the order progresses `PENDING → ANALYZING → RECOMMENDED` within ~60 s. The expanded row shows a `StockAssessment` (2–4 positions with risk flags), a `RoutePlan` (2–4 segments with carrier and cost), and a synthesised summary. The stock assessment and route plan arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the order

**Preconditions:** `StockAnalyst` step timeout set to 1 s (test override).

**Steps:**
1. Submit a demand signal.
2. Watch the order.

**Expected:** the `assessStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the LogisticsPlanner output alone. The order enters `DEGRADED`; the summary notes the missing stock assessment side. No infinite retry.

## J3 — Guardrail blocks an infeasible recommendation

**Preconditions:** Coordinator returns a route plan with a negative `transitDays` value (test fixture).

**Steps:**
1. Submit the fixture SKU.
2. Watch the order.

**Expected:** `guardrailStep` flags the synthesised content; the workflow calls `block`; the order enters `BLOCKED` with a `failureReason` naming the infeasible field. The order row shows the BLOCKED pill; no recommendation is presented as actionable.

## J4 — Eval score appears beside a completed order

**Preconditions:** at least one `RECOMMENDED` order without an `evalScore`.

**Steps:**
1. Wait for `FulfillmentEvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the order row.

**Expected:** the order gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score beside the status pill. Delivery was never delayed by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `DemandSimulator` drips a demand signal from `demand-signals.jsonl` every 60 s; each becomes an order that flows through the full pipeline. The App UI is non-empty on first load without any manual submissions.

## J6 — Multiple concurrent demand signals

**Preconditions:** service running; no prior orders.

**Steps:**
1. Submit three demand signals in rapid succession (different SKUs).
2. Observe all three order rows.

**Expected:** three separate `OptimizationWorkflow` instances run concurrently, each keyed by its own `orderId`. All three progress independently through the PENDING → ANALYZING → RECOMMENDED path without interference. No briefId collision or shared state across orders.
