# User journeys — fomc-event-analyst

## J1 — Submit a seeded FOMC event and get a policy analysis

**Preconditions:** Service running on declared port (`http://localhost:9498/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded event `2026-Q2-rate-decision` has a matching `src/main/resources/sample-data/market/2026-Q2-rate-decision.json` file.

**Steps:**
1. Open `http://localhost:9498/` → App UI tab.
2. From the **Pick a seeded event** dropdown, pick `2026-Q2 rate decision`.
3. Click **Run analysis**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `GATHERING` within 1 s more.
- Within ~20 s the card reaches `GATHERED`. The right pane shows the Market Snapshot table with ≥ 4 rows; each row has a non-empty indicator name, numeric value, and unit.
- Within ~20 s more the card reaches `INTERPRETED`. The right pane shows ≥ 2 policy signals; every `signal.groundedIndicatorId` matches an `indicator.indicatorId` in the snapshot table.
- Within ~20 s more the card reaches `SYNTHESIZED`, then `REVIEWED` within 1 s of that. The right pane shows a `PolicyAnalysis` with `sections.length == signals.length`, every section has at least one grounding ref, every `groundedIn.indicatorId` matches a snapshot indicator, and the verdict badge shows ACCEPTED.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Financial output guardrail blocks an ungrounded analysis

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `synthesize-analysis.json` includes one entry whose first `PolicySection.groundedIn` references an `indicatorId` absent from the paired `MarketSnapshot`.

**Steps:**
1. Submit any seeded FOMC event three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/analyses/sse`).

**Expected:**
- On the third submission's `synthesizeStep`, the agent's first candidate `PolicyAnalysis` references an indicator not in the recorded `MarketSnapshot`. `FinancialOutputGuardrail` rejects it; a `GuardrailRejected{phase: "SYNTHESIZE", reason: "financial-quality-violation: market-grounding: ..."}` event lands on the entity.
- The rejected candidate is never committed as `AnalysisSynthesized` — there is no event log entry for it.
- The agent's second iteration produces a correctly grounded `PolicyAnalysis`. The card eventually reaches `REVIEWED` with verdict `ACCEPTED`.
- The card in the App UI shows the small red dot indicating a rejection fired. The review-log strip on the right pane shows the one rejected attempt with its full structured reason, followed by the accepted verdict.

## J3 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock).

**Steps:**
1. Submit any seeded FOMC event.
2. Wait for `REVIEWED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the analysisId.

**Expected:**
- The GATHER task's log entries show only `fetchIndicators` and `fetchYieldCurve` calls.
- The INTERPRET task's log entries show only `extractPolicySignals` and `classifyRateMove` calls.
- The SYNTHESIZE task's log entries show only `draftPolicySection` and `compileGrounding` calls.
- No cross-phase tool calls appear.
- The order of tasks in the log is GATHER → INTERPRET → SYNTHESIZE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J4 — Empty sections rejected by guardrail

**Preconditions:** Mock LLM mode. Configure one `synthesize-analysis.json` entry to return a `PolicyAnalysis` with `sections = []`.

**Steps:**
1. Submit any seeded event.
2. Watch until the card's status stabilises.

**Expected:**
- The candidate `PolicyAnalysis` with empty sections triggers the guardrail's non-emptiness check.
- `GuardrailRejected{phase: "SYNTHESIZE", reason: "financial-quality-violation: non-empty-sections: sections list is empty"}` lands on the entity.
- The review-log strip on the card shows `REJECTED-AND-RETRIED` as the intermediate state.
- The agent's retry produces a non-empty `PolicyAnalysis`; the card reaches `REVIEWED` with verdict `ACCEPTED`.

## J5 — Custom event with no matching market data file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/market/<custom-event-slug>.json` exists for the user's event.

**Steps:**
1. In the App UI, type a custom event identifier (e.g. `hypothetical-emergency-meeting`) into the event field.
2. Click **Run analysis**.

**Expected:**
- `GatherTools.fetchIndicators` returns an empty list; `fetchYieldCurve` returns a zero-valued snapshot.
- The agent's GATHER task returns a `MarketSnapshot` with `indicators = []`.
- The workflow advances to `interpretStep`. The INTERPRET task returns a `PolicySignalSet` with `signals = []` and a `RateMoveForecast` with `basisPoints = 0` and a narrative explaining the gap.
- The workflow advances to `synthesizeStep`. The SYNTHESIZE task returns a `PolicyAnalysis` with `eventName = "(no policy signals)"` and `sections = []`.
- The guardrail accepts the empty sections on the grounds that there are no signals to cover — the non-emptiness check passes vacuously when `PolicySignalSet.signals` is also empty.
- The card reaches `REVIEWED` with verdict `ACCEPTED`. The rate outlook chip shows `hold (no data)`. Nothing crashes; the empty analysis is honestly empty.

## J6 — Signal count equals section count

**Preconditions:** Service running. Any model provider. Seeded event `2025-Q4-balance-sheet-runoff` whose mock-paired `PolicySignalSet` carries 3 signals.

**Steps:**
1. Submit the seeded event `2025-Q4 balance sheet runoff`.
2. Wait for `REVIEWED`.

**Expected:**
- The recorded `PolicyAnalysis.sections.length` equals the recorded `PolicySignalSet.signals.length` (both 3).
- Every `section.signalId` matches one `signal.signalId` from the signal set, one-to-one.
- The verdict badge shows ACCEPTED and the review-log strip shows no rejections.
- If the agent's first attempt produced 2 sections instead of 3 (silent collapse), the guardrail would have rejected it with a signal-attribution failure (the missing signal has no matching section). The presence of ACCEPTED with 3 sections confirms parity held.
