# User journeys — workflow-observability

## J1 — Workload item arrives and is processed

**Preconditions:** Service running on port 9281; valid model-provider API key set; `WorkloadPoller` enabled.

**Steps:**
1. Open `http://localhost:9281/` -> App UI tab.
2. Wait up to 20 s for the first simulated workload item.

**Expected:**
- Item appears with status QUEUED, then within 1 s transitions to ROUTED with a routing chip showing SUMMARISE, ESCALATE, or SKIP.
- For SUMMARISE items: status reaches PROCESSING, then EXPORTED within 30 s. The detail pane shows the routing rationale and the SummaryAgent output.
- The span waterfall shows at least one span for RouterAgent. SUMMARISE items show a second span for SummaryAgent.

## J2 — Span telemetry is visible and accurate

**Preconditions:** At least one item in EXPORTED status.

**Steps:**
1. Select an EXPORTED item in the item list.
2. Inspect the span waterfall in the detail pane.

**Expected:**
- At least two spans are present for SUMMARISE items: `RouterAgent.route` and `SummaryAgent.process`.
- Each span shows a non-zero `durationMs`, non-zero `promptTokens`, and `status: OK`.
- Span start/end times are chronologically ordered and non-overlapping for sequential steps.

## J3 — ESCALATE routing terminates without SummaryAgent

**Preconditions:** An item in the simulator's workload-items.jsonl with routing category ESCALATE.

**Steps:**
1. Wait for a workload item that triggers ESCALATE routing (the simulator cycles through all types).
2. Observe the item in the App UI.

**Expected:**
- Item status reaches ESCALATED without passing through PROCESSING.
- The span waterfall shows exactly one span: `RouterAgent.route`. No `SummaryAgent.process` span.
- No `SummaryResult` is visible in the detail pane.

## J4 — Eval score appears after EvalSampler runs

**Preconditions:** At least one EXPORTED item with no `evalScore`. EvalSampler schedule reduced for test (`EVAL_SAMPLER_SECONDS=60`).

**Steps:**
1. Confirm at least one item is in EXPORTED status.
2. Wait 60 s.

**Expected:**
- The item's status transitions to EVALUATED.
- The detail pane shows an eval score chip (1-5) and the one-sentence rationale.
- The `/api/traces/{id}` response includes `eval.score` populated.

## J5 — Custom item submitted via API follows the same pipeline

**Preconditions:** Service running.

**Steps:**
1. Submit: `POST /api/traces` with body `{ "workloadType": "policy-doc-review", "payload": "Access control policy v3.2 reviewed 2026-06-01. Approved by CISO. Next review: 2027-06-01.", "sourceTag": "manual-submit" }`.
2. Note the returned `itemId`.
3. Poll `GET /api/traces/{itemId}` until status is EXPORTED.

**Expected:**
- Item appears in the App UI trace feed alongside polled items.
- Routing decision and summary result are populated.
- Spans show the same structure as polled items.

## J6 — Trace export to built-in log (default backend)

**Preconditions:** `TRACE_BACKEND` env var is unset (default mode); `LOG_LEVEL=INFO`.

**Steps:**
1. Observe service logs after an item reaches EXPORTED.

**Expected:**
- Each span produces an INFO log line containing `spanId`, `operationName`, `durationMs`, `promptTokens`, `completionTokens`, and `status`.
- No HTTP calls are made to Arize Phoenix or Langfuse endpoints.
- If `TRACE_BACKEND=arize` and `ARIZE_PHOENIX_URL` is set, an HTTP POST to that URL is attempted instead.
