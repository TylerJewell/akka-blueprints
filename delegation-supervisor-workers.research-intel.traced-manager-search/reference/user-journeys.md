# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a query and watch the full traced run

**Preconditions:** service running on `http://localhost:9305/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a query (e.g., "What evaluation frameworks exist for open-source LLMs?") and click Submit.
2. Observe the new run row via SSE; watch status transitions.
3. After TRACED, click the row to expand.

**Expected:** the run progresses `DISPATCHED → SEARCHING → TRACED` within ~90 s. The expanded row shows a `SearchResultBundle` (3–5 page results), a trace summary bar chart with per-step token counts, a `TraceReport` naming the hotStep, and a synthesised answer of 60–150 words with source URLs.

## J2 — URL allow-list guardrail blocks a prohibited fetch

**Preconditions:** service running; SearchAgent receives a page result pointing to a domain not in the allow-list (inject via the mock fixture or by temporarily removing a domain from the config list).

**Steps:**
1. Submit a query whose mocked search results include an out-of-list URL.
2. Watch the run row.

**Expected:** the run enters `BLOCKED_TOOL`; the row shows the offending URL in a red callout. No HTTP request to the blocked domain is made. The workflow does not retry or fall back silently.

## J3 — MaxPages exceeded degrades to a partial run

**Preconditions:** set `maxPages` to 2 in the SearchPlan fixture; mock search returns 6 results.

**Steps:**
1. Submit a query.
2. Watch the run row.

**Expected:** SearchAgent stops after 2 pages and signals completion. The workflow transitions to `PARTIAL`. A synthesised answer is produced from the 2 available results and notes the search was incomplete. Token counts in the trace reflect only the 2 pages visited.

## J4 — Eval score appears beside a traced run

**Preconditions:** at least one `TRACED` run without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly.
2. Observe the run row in the App UI.

**Expected:** the run gains an `evalScore` (1–5) and an `evalRationale`; the App UI row displays the score beside the answer. Answer delivery was not blocked waiting for the eval.

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `QuerySimulator` drips a query from `search-queries.jsonl` every 60 s; each becomes a run that flows through the full pipeline. The App UI shows non-empty run history on first load, with at least one `TRACED` run visible after 3 minutes.
