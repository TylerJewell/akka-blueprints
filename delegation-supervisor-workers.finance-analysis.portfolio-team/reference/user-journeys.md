# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a holdings set and watch parallel analysis

**Preconditions:** service running on `http://localhost:9528/`; a model provider configured (real or mock); sector tag is a permitted value (e.g., "technology").

**Steps:**
1. Open the App UI tab. Enter a `portfolioId`, `sector`, and one or more holdings lines. Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `PLANNING → IN_PROGRESS → CONSOLIDATED` within ~60 s. The expanded row shows a `PositionAssessment` (3–6 position notes with risk levels), a `MarketContext` (2–4 headwinds, 2–4 tailwinds), and a consolidated executive summary (100–160 words). Holdings assessment and market context arrive close together because the workers ran in parallel.

## J2 — Prohibited sector is rejected before any agent work

**Preconditions:** service running. Submit with a sector tag that is on the prohibited list (e.g., "weapons").

**Steps:**
1. Open the App UI tab. Enter `sector = weapons`. Submit.

**Expected:** the endpoint returns a 400 with `{ "error": "sector 'weapons' is not permitted", "code": "SECTOR_PROHIBITED" }`. No report row appears in the live list. No AutonomousAgent was invoked.

## J3 — Worker timeout degrades the report

**Preconditions:** `HoldingsAnalyst` step timeout set to 1 s (test override).

**Steps:**
1. Submit a valid holdings set.
2. Watch the report.

**Expected:** the `assessStep` times out, the workflow routes to `degradeStep`, and the Coordinator consolidates from the MarketContext alone. The report enters `DEGRADED`; the executive summary notes the missing holdings assessment. No infinite retry.

## J4 — Guardrail blocks a report with an explicit investment directive

**Preconditions:** Coordinator is configured (via test fixture) to return a `ConsolidatedReport` containing the phrase "buy AAPL".

**Steps:**
1. Submit the fixture holdings set.
2. Watch the report.

**Expected:** `guardrailStep` flags the consolidated content; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason`. The report is never surfaced as a finished result in the CONSOLIDATED state.

## J5 — Eval score appears beside a consolidated report

**Preconditions:** at least one `CONSOLIDATED` report without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Refresh the report row.

**Expected:** the report gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Report delivery was never delayed by the eval (non-blocking).

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `HoldingsSimulator` drips a holdings entry from `sample-holdings.jsonl` every 60 s; each becomes a report that flows through the full pipeline (sanitize → plan → parallel fan-out → consolidate → guardrail). The App UI is non-empty on first load.
