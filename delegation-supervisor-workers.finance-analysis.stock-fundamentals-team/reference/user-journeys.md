# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a ticker and watch three-way parallel synthesis

**Preconditions:** service running on `http://localhost:9437/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a ticker (e.g., `AAPL`) and Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `QUEUED → ANALYSING → COMPLETE` within ~90 s. The expanded row shows `FinancialMetrics` (revenue growth, EPS, operating margin), `NewsSummary` (3–6 headlines with sentiments), `RatioSet` (P/E, P/B, ROE, debt/equity), and a synthesised recommendation with a stance and a disclaimer. Financials, news, and ratios arrive close together because the three workers ran in parallel.

## J2 — Worker timeout degrades the report

**Preconditions:** `FinancialsAgent` step timeout set to 1 s (test override).

**Steps:**
1. Submit a ticker.
2. Watch the report.

**Expected:** the `financialsStep` times out; the workflow routes to `degradeStep` and the Coordinator synthesises from the NewsAgent and RatioAgent outputs alone. The report enters `DEGRADED`; the recommendation summary notes the missing financials input. No infinite retry. The two remaining worker results are visible in the expanded row.

## J3 — Guardrail blocks a direct-advice recommendation

**Preconditions:** Coordinator returns a recommendation containing a direct buy/sell instruction without a disclaimer (test fixture or prompt injection).

**Steps:**
1. Submit the fixture ticker.
2. Watch the report.

**Expected:** `guardrailStep` flags the synthesised content; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason` identifying the guardrail violation. The recommendation is never shown in the App UI list's done states.

## J4 — Eval score appears beside a complete report

**Preconditions:** at least one `COMPLETE` report without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually via the dev console.
2. Observe the report row.

**Expected:** the report gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Report delivery was never blocked by the eval (non-blocking).

## J5 — Sector sanitizer redacts price-sensitive text and passes the report

**Preconditions:** Coordinator returns a recommendation containing a price-target assertion framed as fact (test fixture).

**Steps:**
1. Submit the fixture ticker.
2. Watch the report.

**Expected:** `sanitizerStep` detects the price-sensitive phrase and either redacts it (if possible) or blocks. If the sanitizer successfully redacts the phrase, the report proceeds to `COMPLETE` with `sanitizerVerdict = "ok"`. If it cannot safely redact, the report enters `BLOCKED`.

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TickerSimulator` drips a ticker from `tickers.jsonl` every 90 s; each becomes a report that flows through the full pipeline. The App UI is non-empty on first load.
