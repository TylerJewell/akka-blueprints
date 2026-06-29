# User journeys — traced-agent-otel

## J1 — Submit a prompt and see a complete trace

**Preconditions:** Service running on declared port (`http://localhost:9621/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9621/` → App UI tab.
2. From the **Prompt preset** dropdown, pick `Code generation`.
3. Leave **Tool mode** off.
4. Click **Run agent**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RUNNING` within 1 s.
- Within 30 s the card reaches `COMPLETED`. The right pane shows the agent's answer and the span summary: `totalSpans ≥ 1`, `llmTurnCount = 1`, `toolCallCount = 0`, `errorCount = 0`, `wallClockMillis > 0`.
- Within 2 s the card transitions to `FLUSHED`. The flush result shows `exporterHealthy = true`, `spansExported = totalSpans`, `spansDropped = 0`, and a clickable Zipkin trace URL.
- Within 1 s the card transitions to `MONITORED`. The performance chip shows a quality score of 4 or 5; p50 latency matches the LLM_CALL span duration; `rationale` is a single sentence.

## J2 — Tool mode produces tool-call spans

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Open App UI tab.
2. From the **Prompt preset** dropdown, pick `Reasoning`.
3. Toggle **Tool mode** on.
4. Click **Run agent**.

**Expected:**
- The conversation reaches `MONITORED`.
- The span tree in the right pane shows at least one `TOOL_CALL` span as a child of the root `LLM_CALL` span. Its `operationName` is `"tool.web_search"` and its `kind` is `TOOL_CALL`.
- `spanSummary.toolCallCount ≥ 1`.
- `performanceMonitorResult.toolSuccessRate` is between 0.0 and 1.0 (not N/A).
- The performance chip quality score accounts for the tool-call span; no penalty if `status = OK`.

## J3 — Export failure recorded without blocking the lifecycle

**Preconditions:** Service running with the mock LLM and `FailingOtlpExporter` stub active (set `akka.javasdk.tracing.exporter = "otlp-failing"` in `application.conf`).

**Steps:**
1. Submit any seeded prompt.
2. Watch the conversation lifecycle in the SSE stream (`/api/conversations/sse`).

**Expected:**
- The conversation still transitions through `COMPLETED → FLUSHED → MONITORED`. The lifecycle is not blocked by the export failure.
- `flushResult.exporterHealthy = false`.
- `flushResult.spansDropped = totalSpans` (all spans were dropped because the exporter was unavailable).
- The UI shows a yellow export-warning chip in the flush result section ("Exporter unavailable").
- The Zipkin trace URL is absent (null or a stub URL indicating failure).
- `monitorResult` is still populated; the quality score reflects the span data that was available in memory (not penalised for export failure alone).

## J4 — Zero-duration LLM spans trigger quality score 1

**Preconditions:** Mock LLM mode. The mock's `answer-prompt.json` contains an entry where all `LLM_CALL` spans have identical `startTime` and `endTime` values (zero duration). This entry is selected for the 4th conversation in a sequence (deterministic by `seedFor`).

**Steps:**
1. Submit three seed prompts (J1 steps × 3).
2. Submit a fourth prompt.

**Expected:**
- The fourth conversation reaches `MONITORED`.
- `monitorResult.qualityScore = 1`.
- `monitorResult.rationale` contains the phrase "zero-duration LLM spans".
- `monitorResult.p50Millis = 0` and `monitorResult.p95Millis = 0`.
- The fourth card's border is highlighted red in the UI (quality score ≤ 2 threshold).

## J5 — SpanSummary aggregates match raw spans

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the `Data extraction` seed prompt with tool mode on.
2. Wait for `MONITORED`.
3. Fetch `GET /api/conversations/{id}`.

**Expected:**
- `agentResponse.spans` contains the raw `SpanRecord` list.
- `spanSummary.totalSpans` equals `agentResponse.spans.length`.
- `spanSummary.llmTurnCount` equals the count of spans with `kind = "LLM_CALL"` in `agentResponse.spans`.
- `spanSummary.toolCallCount` equals the count of spans with `kind = "TOOL_CALL"` in `agentResponse.spans`.
- `spanSummary.errorCount` equals the count of spans with `status = "ERROR"` in `agentResponse.spans`.
- `spanSummary.wallClockMillis` equals the duration between the earliest `startTime` and the latest `endTime` across all spans.
