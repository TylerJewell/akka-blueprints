# User journeys — guardrails-baseline

## J1 — Submit a finance message and get a decision

**Preconditions:** Service running on declared port (`http://localhost:9680/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9680/` → App UI tab.
2. From the **Policy set** dropdown, pick `finance (4 rules)`.
3. Click **Load seeded example** to fill the message title and the message textarea.
4. Click **Submit for moderation**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted message; the PII category chips show the categories found (e.g., `email` if the seed message contains one).
- Within 30 s the card reaches `DECISION_RECORDED`. The right pane shows: a verdict badge (ALLOW / BLOCK / ESCALATE), the rationale paragraph, and a rule-result row for *each of the 4 submitted rules*. Every rule result has a non-empty explanation and a confidence value in `[0.0, 1.0]`.
- Within 1 s of `DECISION_RECORDED`, the card reaches `AUDITED` and shows an audit score chip (1–5) plus a one-line note.

## J2 — Input guardrail blocks a forbidden tool invocation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `moderate-message.json` includes an entry that causes the agent to plan a tool invocation not on the allowlist (e.g., `web-search`).

**Steps:**
1. Submit a seeded message three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/moderations/sse`).

**Expected:**
- The third submission's first agent iteration plans a `web-search` tool invocation.
- The `before-agent-invocation` guardrail rejects it. The forbidden tool invocation NEVER results in an actual tool call — the LLM call for that iteration does not happen with `web-search` available.
- The agent loop retries on iteration 2 without the blocked tool and produces a well-formed decision. The card transitions to `DECISION_RECORDED` normally.
- The service log shows one `guardrail.reject` line naming `web-search` as the blocked tool.

## J3 — Output guardrail blocks a malformed decision

**Preconditions:** Service running with the mock LLM selected. The mock includes a deliberately malformed decision entry (a rule result referencing an unknown `ruleId`, or an ALLOW verdict despite a BLOCK rule result).

**Steps:**
1. Submit any seeded message until the mock selects a malformed entry (every 3rd submission by modulo seed).
2. Watch the submission's lifecycle in the network panel (`/api/moderations/sse`).

**Expected:**
- The first agent iteration produces a malformed decision.
- The `after-llm-response` guardrail rejects it. The malformed decision NEVER lands in `ModerationEntity` — there is no `DecisionRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 and produces a well-formed decision. The card transitions to `DECISION_RECORDED` with a decision that satisfies all five output-guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a message containing the literal strings `john.smith@example.com` and `+1-800-555-0199`.
2. Wait for `DECISION_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/moderations/{id}` and read `request.rawMessage`.

**Expected:**
- The logged LLM call body contains only the redacted forms: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`. The raw strings do not appear.
- `request.rawMessage` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `phone` (at minimum).

## J5 — Rule-result completeness

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit any seeded message with the `healthcare (5 rules)` policy set.
2. Wait for `AUDITED`.

**Expected:**
- The decision's `ruleResults` array has exactly 5 entries, one per submitted policy rule.
- Each `ruleResults[i].ruleId` matches one of the 5 submitted `ruleId` values, one-to-one.
- If the agent's first iteration omitted a rule, the output guardrail would have rejected it — the absence of any rejection in the log confirms the first iteration was complete.

## J6 — Verdict consistency enforced by output guardrail

**Preconditions:** Service running with mock LLM. The mock includes an entry where `verdict = ALLOW` but one rule result carries `action = BLOCK` (a consistency violation).

**Steps:**
1. Trigger the inconsistent mock entry by submitting until the modulo-3 seed selects it.
2. Observe the SSE stream and service log.

**Expected:**
- The output guardrail's consistency check fires: `verdict ALLOW is inconsistent with rule result action BLOCK for ruleId <id>`.
- The agent retries. The second iteration returns a decision where `verdict = BLOCK` (or higher), consistent with the rule result.
- No `DecisionRecorded` event carries an inconsistent decision.
