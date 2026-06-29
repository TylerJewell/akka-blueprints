# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a market question and watch parallel synthesis

**Preconditions:** service running on `http://localhost:9263/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a market question and Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `FRAMING → IN_PROGRESS → PUBLISHED` within ~60 s. The expanded row shows an `IndicatorBundle` (3–6 indicators), an `EconomicInterpretation` (3–6 implications), and a synthesised summary that ends with the required disclaimer sentence. Indicators and interpretation arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the report

**Preconditions:** `DataCollector` step timeout set to 1 s (test override).

**Steps:**
1. Submit a market question.
2. Watch the report.

**Expected:** the `collectStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the Economist output alone. The report enters `DEGRADED`; the summary notes the missing indicator side. No infinite retry.

## J3 — Guardrail blocks a report with a speculative claim

**Preconditions:** EconomicsCoordinator returns a report containing an unsupported "buy" recommendation (test fixture).

**Steps:**
1. Submit the fixture question.
2. Watch the report.

**Expected:** `guardrailStep` flags the synthesised content; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason`. The report is never surfaced as a finished result in the App UI's done state.

## J4 — Eval score appears beside a published report

**Preconditions:** at least one `PUBLISHED` report without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it.
2. Refresh the report row.

**Expected:** the report gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `RequestSimulator` drips a market question from `market-questions.jsonl` every 60 s; each becomes a report that flows through the full pipeline. The App UI is non-empty on first load.
