# User journeys — slack-assistant

## J1 — Send a deploy-status query and get an ANSWER reply

**Preconditions:** Service running on declared port (`http://localhost:9945/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9945/` → App UI tab.
2. From the **Seeded message** dropdown, pick `Deploy status query`.
3. Confirm channel name is `deploys` and click **Send to agent**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the sanitized message text; the secret-category chips row is empty (no secrets in this seed message).
- Within 30 s the card reaches `REPLY_RECORDED`. The right pane shows: an `ANSWER` action badge and a non-empty `responseText` of 1–3 sentences. No `runbookUrl` chip.
- Within 1 s of `REPLY_RECORDED`, the card reaches `AUDITED` and shows a `1 / 0` candidate/rejected chip.

## J2 — Guardrail blocks a reply containing a Slack token

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `reply-to-message.json` includes a deliberately malformed entry whose `responseText` contains `xoxb-FAKESLACKTOKEN`.

**Steps:**
1. Send the same seeded message three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/messages/sse`).

**Expected:**
- The third submission's first agent iteration produces a reply containing `xoxb-FAKESLACKTOKEN`.
- The `before-agent-response` guardrail rejects it. The malformed reply NEVER lands in `MessageEntity` — there is no `ReplyRecorded` event with the token-containing payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a clean reply. The card transitions to `REPLY_RECORDED` with a `responseText` free of any secret pattern.
- The service log shows one `guardrail.reject` line per rejected iteration, naming the failed check (`secret-pattern-in-response-text`).

## J3 — Secrets are stripped from message context before the agent sees it

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the task attachment is logged. Any model provider — the sanitizer runs either way.

**Steps:**
1. In the App UI's Message text textarea, paste a message containing `xoxb-T00000000-FAKETOKEN`, `ghp_aaaabbbbccccddddeeee`, and the text `Connection refused on port 5432`.
2. Wait for `REPLY_RECORDED`.
3. Inspect the service log for the task attachment payload (`debug:agent.task.attachment`).
4. Fetch `GET /api/messages/{id}` and read `context.triggerText`.

**Expected:**
- The logged task attachment contains only `[REDACTED-SLACK-TOKEN]`, `[REDACTED-GH-PAT]`, and the non-sensitive text. Neither raw token appears.
- `context.triggerText` in the JSON response still contains the original tokens — the entity preserves them for audit.
- `sanitized.secretCategoriesFound` lists `slack-token` and `gh-pat` (at minimum).

## J4 — ESCALATE classification posts the correct action badge

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Seeded message** dropdown, pick `Incident triage — high alert volume`.
2. Click **Send to agent**.
3. Wait for `AUDITED`.

**Expected:**
- The reply's `actionType` is `ESCALATE`.
- The live list card shows an `ESCALATE` badge (yellow).
- The right-pane reply section shows the escalation-prefixed `responseText` (begins with "Escalating to" or "This needs").
- No `runbookUrl` chip appears.

## J5 — RUNBOOK_REF includes a link chip in the UI

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Seeded message** dropdown, pick `Runbook lookup — database failover`.
2. Click **Send to agent**.
3. Wait for `AUDITED`.

**Expected:**
- The reply's `actionType` is `RUNBOOK_REF`.
- The live list card shows a `RUNBOOK_REF` badge (purple).
- The right-pane reply section shows the runbook-prefixed `responseText` and a link chip beneath it with the `runbookUrl` value.

## J6 — CLARIFY is returned for ambiguous input

**Preconditions:** Service running with the mock LLM selected. A seeded message with ambiguous content triggers the CLARIFY mock entry.

**Steps:**
1. From the **Seeded message** dropdown, pick `Ambiguous alert — needs clarification`.
2. Click **Send to agent**.
3. Wait for `AUDITED`.

**Expected:**
- The reply's `actionType` is `CLARIFY`.
- The live list card shows a `CLARIFY` badge (blue).
- The right-pane reply section shows a `responseText` phrased as a single specific question (ends with `?`).
- No `runbookUrl` chip appears.
