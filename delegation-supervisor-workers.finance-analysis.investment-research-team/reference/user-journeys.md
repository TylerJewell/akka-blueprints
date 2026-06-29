# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a research request and watch parallel synthesis

**Preconditions:** service running on `http://localhost:9801/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a ticker (e.g., `NVDA`) and a question (e.g., "What is the revenue growth trend and current market sentiment?"). Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `PLANNING → IN_PROGRESS → PUBLISHED` within ~60 s. The expanded row shows a `FundamentalsBundle` (3–5 facts with metrics and periods), a `SentimentBundle` (2–4 signals with direction labels), and a synthesised summary. Fundamentals and sentiment arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the report

**Preconditions:** `EquityAnalyst` step timeout set to 1 s (test override).

**Steps:**
1. Submit a ticker and question.
2. Watch the report.

**Expected:** the `fundamentalsStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the MarketScout output alone. The report enters `DEGRADED`; the summary notes the missing fundamentals side. No infinite retry.

## J3 — Sector sanitizer blocks prohibited content

**Preconditions:** Coordinator returns a report summary containing a fabricated price target (test fixture — e.g., "NVDA will reach $2000 by Q4 2025").

**Steps:**
1. Submit the fixture ticker.
2. Watch the report.

**Expected:** `sanitizerStep` flags the synthesised content as a prohibited price prediction; the workflow calls `blockReport`; the report enters `BLOCKED` with a `failureReason` naming the rule that fired. The report is never shown as PUBLISHED in the App UI list.

## J4 — Eval score appears beside a published report

**Preconditions:** at least one `PUBLISHED` report without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly.
2. Observe the report row.

**Expected:** the report gains an `evalScore` (1–5) and an `evalRationale` assessing numeric-claim traceability; the App UI row shows the score. Report delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `RequestSimulator` drips a ticker research request from `research-requests.jsonl` every 60 s; each becomes a report that flows through the full pipeline. The App UI is non-empty on first load.

## J6 — Both workers complete and fundamentals are attributed

**Preconditions:** service running with a real or mock model provider; mock responses include sourced facts.

**Steps:**
1. Submit any ticker.
2. Wait for the report to reach `PUBLISHED`.
3. Expand the report row.

**Expected:** the `FundamentalsBundle` shows at least one fact with a non-empty `source` field. The synthesised summary references only figures present in the facts list — no invented numbers. The `sanitizerVerdict` field on the expanded report is `"ok"`.
