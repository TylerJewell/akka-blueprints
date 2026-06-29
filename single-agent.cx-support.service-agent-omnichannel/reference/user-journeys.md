# User journeys — service-agent-omnichannel

## J1 — Billing dispute handled on WEB channel

**Preconditions:** Service running on declared port (`http://localhost:9739/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9739/` → App UI tab.
2. From the **Channel** dropdown, pick `Web`.
3. From the **Scenario** dropdown, pick `billing dispute`.
4. Click **Load seeded example** to fill the message textarea and Customer ID.
5. Click **Send message**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted message; PII category chips are shown if the seed message contains identifiers.
- The card transitions to `TRIAGING` and the triage chip shows `BILLING`.
- Within 30 s the card reaches `REPLIED`. The right pane shows: a resolution badge (RESOLVED / NEEDS_FOLLOW_UP / ESCALATE), the reply text formatted as `chat-markdown`, and at least one entry in `crmWritesApplied`.
- Within 1 s of `REPLIED`, the eval score chip appears (1–5) with a one-line rationale.

## J2 — Reply guardrail blocks a policy-violating draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `handle-support-case.json` includes deliberately malformed entries (one with `channelFormat` set to an invalid value; one with empty `replyText`).

**Steps:**
1. Send any seeded message three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/cases/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed reply.
- The `before-agent-response` guardrail rejects it. The malformed reply NEVER lands in `CaseEntity` — there is no `ReplySent` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a conforming reply. The card transitions to `REPLIED` with a reply that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — CRM write guardrail corrects an invalid tool call

**Preconditions:** Service running with the mock LLM selected. Mock includes a response where the agent's first `case.create` tool call carries `priority = "EXTREME"` (not in the allowed set).

**Steps:**
1. Send the seeded `technical-fault` message on the WHATSAPP channel.
2. Wait for `REPLIED`.

**Expected:**
- The first `case.create` tool-call attempt is blocked by `CrmWriteGuardrail` with a `tool-rejected` code naming the invalid priority.
- The agent corrects the payload to `priority = NORMAL` and retries the tool call.
- The case card eventually reaches `REPLIED`. The `crmWritesApplied` list shows exactly one `case.create` entry (not two).
- The service log shows one `guardrail.tool-rejected` line for the blocked attempt.

## J4 — HITL escalation transfers case to human queue

**Preconditions:** Service running. Any model provider (real or mock).

**Steps:**
1. In the **Scenario** dropdown, pick `account closure`.
2. Type (or load) a message that includes the phrase "I want to close my account and speak to someone".
3. Click **Send message**.
4. Wait for the card to settle.

**Expected:**
- The agent returns `resolutionIntent = ESCALATE`.
- The case transitions from `HANDLING` → `REPLIED` → `ESCALATED` in quick succession (visible on the SSE stream).
- The App UI case card shows an amber human-queue chip ("Transferred to human agent queue") and no further agent replies are generated.
- `GET /api/cases/{id}` returns `"status": "ESCALATED"`.
- The eval score chip still appears; escalating a genuine account-closure request scores ≥ 3.

## J5 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider.

**Steps:**
1. Send a message containing the literal strings `jane.smith@example.com`, `07700 900123`, and `ACC-9982341`.
2. Wait for `REPLIED` or `ESCALATED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/cases/{id}` and read `message.rawMessage`.

**Expected:**
- The logged LLM call body contains only redacted forms: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-ACCOUNT]`. The raw strings do not appear anywhere in the log.
- `message.rawMessage` in the JSON response still contains the raw strings — the audit record is preserved.
- `sanitized.piiCategoriesFound` lists `email`, `phone`, `account-number` (at minimum).

## J6 — Channel-format mismatch on VOICE channel

**Preconditions:** Service running with the mock LLM selected. Mock includes a response entry where `channelFormat = "chat-markdown"` for a VOICE channel case (a deliberate mismatch).

**Steps:**
1. Select channel `Voice`, scenario `technical fault`.
2. Click **Load seeded example** and **Send message**.

**Expected:**
- The guardrail rejects the first draft because `channelFormat = "chat-markdown"` does not match the expected `voice-ssml` for a VOICE channel case.
- The agent retries with `channelFormat = "voice-ssml"` and a plain-language reply suitable for speech.
- The final card shows `REPLIED` with `channelFormat = voice-ssml` visible in the detail pane.
