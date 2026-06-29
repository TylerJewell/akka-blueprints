# User journeys — feed-monitor

## J1 — Feed item arrives and is posted to Slack

**Preconditions:** Service running on declared port; valid model-provider API key set; `FeedPoller` enabled; `feed-monitor.slack.allowed-channels` contains `"#research-intel"`.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 60 s for the first simulated feed item.

**Expected:**
- Item appears with status RECEIVED, then within 1 s transitions to SUMMARIZED. The AI summary and extracted topics are visible.
- Within 2 s the classification chip appears (`NOTIFY` for the fixture's high-signal items).
- Within 5 s the status reaches POSTED. The Slack post record is visible (stub channel, guardrailPassed = true).

## J2 — SUPPRESS classification produces no Slack post

**Preconditions:** Service running; fixture contains at least one SUPPRESS-classified item.

**Steps:**
1. Wait for a SUPPRESS item to appear in the feed list.

**Expected:**
- Item transitions from RECEIVED → SUMMARIZED → CLASSIFIED → SUPPRESSED without triggering a Slack post.
- No `SlackPost` record appears on the item's detail pane.
- `FeedView` row shows status SUPPRESSED; the classification chip shows "SUPPRESS".

## J3 — REVIEW item is overridden by a human

**Preconditions:** At least one PENDING_REVIEW item in the UI.

**Steps:**
1. Select the PENDING_REVIEW item in the feed list.
2. Click Override. A channel input field appears pre-filled with `"#research-intel"`.
3. Click Confirm.

**Expected:**
- Item transitions from PENDING_REVIEW → NOTIFY_REQUESTED → POSTED.
- The `SlackPost` record shows `decidedBy` populated from the UI's session id.
- The detail pane shows the override decision and the resulting post record.

## J4 — Guardrail blocks a post to a disallowed channel

**Preconditions:** Service running; `feed-monitor.slack.allowed-channels` does NOT contain `"#general"`.

**Steps:**
1. Issue `POST /api/feed/{id}/override` with body `{ "decidedBy": "analyst-01", "targetChannel": "#general" }` for a PENDING_REVIEW item.

**Expected:**
- Item transitions from PENDING_REVIEW → NOTIFY_REQUESTED, then immediately → FAILED.
- The entity's `failureReason` is `"guardrail:channel-not-allowed"`.
- No Slack post record appears; `guardrailPassed` is absent (no stub call was made).

## J5 — Eval score appears after EvalRunner ticks

**Preconditions:** At least one POSTED item with no `evalScore`; EvalRunner interval reduced to 60 s via `EVAL_RUNNER_SECONDS=60`.

**Steps:**
1. Post an item (see J1).
2. Wait 60 s.

**Expected:**
- The feed card shows an eval score chip (1–5).
- The detail pane shows the score and one-sentence rationale.
- `GET /api/feed/{id}` returns `evalScore` and `evalRationale` populated.
