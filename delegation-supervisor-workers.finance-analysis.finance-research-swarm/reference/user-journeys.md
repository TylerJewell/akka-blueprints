# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a ticker query and watch the swarm deliver an integrated report

**Preconditions:** service running on `http://localhost:9351/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter "Apple" in the query field and Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `RESOLVING → IN_PROGRESS → PUBLISHED` within ~90 s. The resolved ticker shows "AAPL / NASDAQ". The expanded row shows a `PriceSummary` (5 price points), a `NewsSummary` (3–5 headlines), a `FundamentalsSnapshot` (peRatio, eps, marketCapUsd, revenueGrowthPercent), and an integrated summary. Price, news, and fundamentals arrive close together because the three data workers ran in parallel.

## J2 — Worker timeout degrades the report to PARTIAL

**Preconditions:** `PriceAnalyst` step timeout set to 1 s (test override).

**Steps:**
1. Submit a ticker query.
2. Watch the report.

**Expected:** `priceStep` times out; the workflow routes to `partialStep`; the Coordinator integrates from `NewsSummary` and `FundamentalsSnapshot` only. The report enters `PARTIAL`; the integrated summary notes the missing price data in one sentence. No infinite retry.

## J3 — Sector sanitizer blocks a non-compliant report phrase

**Preconditions:** coordinator returns an integrated summary containing "guaranteed return" (test fixture).

**Steps:**
1. Submit the fixture query.
2. Watch the report.

**Expected:** `sanitizerStep` detects the prohibited phrase; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason` identifying the phrase. The report is never surfaced as `PUBLISHED` in the App UI list.

## J4 — Output guardrail blocks a report with fabricated fundamentals

**Preconditions:** coordinator returns an `IntegratedReport` whose `peRatio` in the summary contradicts the `FundamentalsSnapshot` payload (test fixture).

**Steps:**
1. Submit the fixture query.
2. Watch the report after the sanitizerStep passes.

**Expected:** `guardrailStep` flags the numeric inconsistency; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason`. Delivery is blocked even when the sanitizerStep passed.

## J5 — Eval score appears beside a published report

**Preconditions:** at least one `PUBLISHED` report without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Refresh the report row.

**Expected:** the report gains an `evalScore` (1–5) and an `evalRationale` explaining which numeric claims were checked; the App UI row shows the score. Report delivery was never blocked by the eval (non-blocking).

## J6 — Background simulator supplies reports without user interaction

**Preconditions:** no UI interaction after service start.

**Steps:**
1. Leave the service running for 2+ minutes.

**Expected:** `TickerSimulator` drips one query from `ticker-queries.jsonl` every 60 s; each becomes a report that flows through the full pipeline. The App UI is non-empty on first load and shows at least two reports in various lifecycle states.
