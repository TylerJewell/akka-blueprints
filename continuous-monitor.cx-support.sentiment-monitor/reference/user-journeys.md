# User journeys — sentiment-monitor

## J1 — Comment arrives and gets scored

**Preconditions:** Service running on declared port; valid model-provider API key set or mock LLM selected; `CommentPoller` enabled.

**Steps:**
1. Open `http://localhost:9155/` → App UI tab.
2. Wait up to 20 s for the first simulated comment.

**Expected:**
- A thread card appears (or updates) on the left with the issue ID and title.
- Within 3 s the card shows a sentiment score chip for the new comment.
- The running average and consecutive-negative count update.
- The SSE stream delivers a `thread-update` event visible in browser DevTools.

## J2 — Thread reaches CRITICAL and triggers an alert

**Preconditions:** Issue thread `ENG-SIM-01` (seeded in `comments.jsonl`) is set to produce 6 consecutive negative comments (scores −3 to −5).

**Steps:**
1. Wait for comments on `ENG-SIM-01` to arrive over successive 20 s ticks.
2. Observe the thread card on the left.

**Expected:**
- After the 3rd consecutive negative score the trend badge changes to DECLINING (orange).
- After the 5th (or configured threshold) consecutive negative score the badge changes to CRITICAL (red) and `criticalAt` is set.
- `AlertDispatchWorkflow` starts: guardrail passes (channel `C_ESCALATIONS` is in the allowlist), `TrendAnalysisAgent` produces an `AlertSummary`, and the `postSlackAlert` tool stub is called.
- The thread detail shows "Alert sent to #C_ESCALATIONS".
- The service log records `AlertDispatched` with the alert summary.

## J3 — Guardrail blocks dispatch to an unauthorized channel

**Preconditions:** Service running. Manually POST a test comment batch that sets `targetChannel = "C_RANDOM"` (not in allowlist). This can be done by temporarily editing `application.conf`'s `sentiment.alert.allowed-channels` to remove all channels, then triggering a critical trend.

**Steps:**
1. Edit `application.conf` to set `sentiment.alert.allowed-channels = []`.
2. Restart the service.
3. Wait for a thread to reach CRITICAL.

**Expected:**
- `AlertDispatchWorkflow.validateChannelStep` returns `CHANNEL_NOT_ALLOWED`.
- No call to `postSlackAlert` is made (verifiable in service log: no `postSlackAlert invoked` line).
- The workflow terminates cleanly; no exception or crash is logged.
- The thread's `lastAlert.dispatched` remains `false`.

## J4 — User silences an alert

**Preconditions:** A thread in CRITICAL status with `lastAlert.dispatched == true`.

**Steps:**
1. Select the CRITICAL thread in the left panel.
2. Click the Silence button.

**Expected:**
- Thread status changes to SILENCED.
- The trend badge greys out.
- Subsequent comments on the same thread are still scored and the running average updates, but no further `AlertDispatchWorkflow` instances start for this thread until it is resolved.
- `AlertSilenced` event is visible in the entity's event log (via `GET /api/sentiment/threads/{issueId}`).

## J5 — Eval runner surfaces a drift result

**Preconditions:** At least 5 scored comments exist. `SentimentEvalRunner` interval set to 60 s for the test (override via `sentiment.eval.runner.interval-seconds = 60` in application.conf).

**Steps:**
1. Wait 60 s after the first comments are scored.

**Expected:**
- `SentimentEvalRunner` picks up to 10 labeled entries from `ground-truth-labels.jsonl`.
- A `DriftEvalRecorded` event is emitted on `SentimentEvalEntity`.
- The Eval Matrix tab shows the drift verdict badge (NO_DRIFT / MINOR_DRIFT / SIGNIFICANT_DRIFT) and the mean absolute error value.
- The App UI tab's selected thread detail shows the drift badge in the lower section.
