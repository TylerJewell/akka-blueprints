# User journeys — ig-dm-agent

## J1 — Submit a product-availability inquiry and get a brand-safe reply

**Preconditions:** Service running on declared port (`http://localhost:9721/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9721/` → App UI tab.
2. From the **Brand profile** dropdown, pick `Retail`.
3. Click **Load seeded example** to fill the Sender ID field and the message textarea with the product-availability seed DM.
4. Click **Send to agent**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the sanitized message; if the seed DM contains an email or phone token, the PII category chips appear above it.
- Within 30 s the card reaches `REPLIED`. The right pane shows the `replyText`, a tone label (FRIENDLY or PROFESSIONAL for the Retail profile), and a `toneConfidenceScore` of 3 or higher.
- The reply does not contain any term from the Retail brand's `prohibitedTerms` list and does not exceed 280 characters.

## J2 — Guardrail blocks a prohibited-term draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `reply-to-message.json` includes a deliberately malformed entry whose `replyText` contains the prohibited term "guarantee".

**Steps:**
1. Submit any seeded DM three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/messages/sse`).

**Expected:**
- The third submission's first agent iteration produces a draft containing "guarantee".
- The `before-agent-response` guardrail rejects it. The non-compliant draft NEVER lands in `DmEntity` — there is no `ReplyRecorded` event with that text.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a compliant reply. The card transitions to `REPLIED` with a reply that passes all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration, naming the violated rule (e.g., `prohibited-term: guarantee`).

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Paste a DM containing the literal strings `user@example.com` and `+1-555-867-5309`.
2. Set **Brand profile** to any seeded profile and click **Send to agent**.
3. Wait for `REPLIED`.
4. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
5. Fetch `GET /api/messages/{id}` and read `inbound.rawText`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`. The raw strings do not appear in the attachment.
- `inbound.rawText` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `phone` (at minimum).

## J4 — Over-length draft caught by the guardrail

**Preconditions:** Mock LLM mode. The mock's `reply-to-message.json` includes a malformed entry whose `replyText` length exceeds the Retail brand's `maxReplyChars` (280).

**Steps:**
1. Ensure you are using the Retail brand profile.
2. Submit a seeded DM that maps to the over-length mock entry (every 3rd message in mock mode).
3. Wait for the card to reach `REPLIED`.

**Expected:**
- The guardrail rejects the over-length draft. The service log shows `guardrail.reject: reply-exceeds-max-chars`.
- The final `ReplyRecorded` event carries a reply of 280 characters or fewer.
- The card transitions to `REPLIED` normally; the moderator sees no indication that an earlier draft was rejected (the guardrail is internal to the agent loop).

## J5 — Brand profile switch changes tone

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the same order-status complaint DM twice — once with the Retail brand profile, once with the Hospitality brand profile.
2. Compare the tone labels on the two reply cards.

**Expected:**
- The Retail reply carries toneLabel `FRIENDLY` or `PROFESSIONAL`.
- The Hospitality reply carries toneLabel `EMPATHETIC` or `PROFESSIONAL` (hospitality brands lean toward empathy for complaint handling).
- Neither reply contains prohibited terms from its respective brand profile.
- Both replies are within their brand's respective `maxReplyChars`.

## J6 — Failed message preserves partial state

**Preconditions:** Mock LLM mode, with `maxIterationsPerTask` exhaustion simulated by providing 3 consecutive malformed entries for one message seed.

**Steps:**
1. Configure the mock to return malformed entries on all 3 iterations for a specific messageId (edit `mock-responses/reply-to-message.json` to have 3 prohibited-term entries that the seed selects).
2. Submit that seed DM.
3. Wait for the card to reach `FAILED`.

**Expected:**
- The card shows status `FAILED` with a red pill.
- The right pane shows the `sanitized` message and the `inbound` details — the partial state is preserved.
- No `ReplyRecorded` event exists for this message; the entity log ends at `ReplyStarted`.
- The service log shows 3 `guardrail.reject` lines followed by a `workflow.step.failed` line for `replyStep`.
