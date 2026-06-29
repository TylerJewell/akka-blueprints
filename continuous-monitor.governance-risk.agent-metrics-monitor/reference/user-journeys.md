# User journeys — agent-metrics-monitor

## J1 — First batch arrives and health cards render

**Preconditions:** Service running on declared port; valid model-provider API key set or mock LLM configured; `MetricsPoller` enabled.

**Steps:**
1. Open `http://localhost:9322/` → App UI tab.
2. Wait up to 60 s for the first simulated batch.

**Expected:**
- Three agent health cards appear (order-processor, fraud-screener, customer-router), each showing a health status badge.
- The first batch in the simulator is all-HEALTHY; all three cards show the green HEALTHY chip.
- The Architecture tab's stat tiles show 3 Agents, 1 Workflow, 2 ESEs, 1 Consumer, 2 TimedActions, 1 View, 2 HttpEndpoints.

## J2 — Degraded agent detected and displayed

**Preconditions:** J1 complete; at least two batches have been processed (the simulator's third batch contains a DEGRADED fraud-screener).

**Steps:**
1. Wait for the third batch to be polled (up to 3 minutes from start).
2. Observe the fraud-screener card.

**Expected:**
- The fraud-screener card's status chip changes from HEALTHY to DEGRADED.
- The card shows the anomaly signal `"errorRate 0.14 > threshold 0.10"` as a muted chip.
- Clicking the card opens the detail panel with the full `AnomalyResult`, the `recentSamples` history table (showing the rising error rate trend), and the latest narrative excerpt.

## J3 — Critical agent triggers prominent display

**Preconditions:** The simulator's fifth batch contains a CRITICAL order-processor.

**Steps:**
1. Wait for the fifth batch (up to 5 minutes from start).
2. Observe the agent list.

**Expected:**
- The order-processor card moves to the top of the list (CRITICAL agents sort first).
- The card's status chip is red.
- The anomaly signals `"errorRate 0.27 > threshold 0.20"` and `"p99LatencyMs 9500 > threshold 8000"` are visible on the card.
- The detail panel's metric history table shows the worsening trend across the last two samples.

## J4 — Summary narrative appears in dashboard

**Preconditions:** At least one batch has been processed.

**Steps:**
1. Open the App UI tab.
2. Look at the summary strip below the health cards.

**Expected:**
- A `HealthSummary` paragraph is visible, produced by `SummaryNarratorAgent`.
- The text cites the correct agent counts (matching the batch's actual CRITICAL/DEGRADED counts).
- The summary does not name any agent not present in the monitored set.

## J5 — Eval score appears on summary

**Preconditions:** At least one summary exists with no eval score. For local testing, reduce `EvalRunner` to 60 s via `EVAL_RUNNER_SECONDS=60`.

**Steps:**
1. Wait 60 s after a summary is produced.

**Expected:**
- The summary card shows a score chip (1–5).
- The detail panel shows the eval rationale sentence.
- `GET /api/metrics/agents/{agentId}` returns `latestEvalScore` populated.

## J6 — Recovery: all-healthy batch resets status

**Preconditions:** At least one agent is currently DEGRADED or CRITICAL.

**Steps:**
1. The simulator's next batch has all agents HEALTHY (batches cycle; a HEALTHY batch follows the CRITICAL one).
2. Wait up to 60 s for the batch to be polled and processed.

**Expected:**
- The previously DEGRADED/CRITICAL agent's card transitions to HEALTHY.
- Its anomaly signals are cleared.
- The `MetricsView` SSE stream emits an `agent-health-update` event with `currentStatus: "HEALTHY"`.
