# User journeys — rag-support-bot

## J1 — Submit an order-status query and receive a grounded reply

**Preconditions:** Service running on declared port (`http://localhost:9423/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The four seeded knowledge-base articles are loaded.

**Steps:**
1. Open `http://localhost:9423/` → App UI tab.
2. In the **Topic filter** dropdown, pick `Orders`.
3. Click **Load seeded example** to fill the textarea with the order-status seed query.
4. Click **Send**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted query; PII category chips show `order-id` and `email` (from the seed query).
- Within ~2 s the card transitions to `RETRIEVING` then `REPLYING`.
- Within 30 s the card reaches `REPLY_RECORDED`. The right pane shows: an outcome badge (ANSWERED / PARTIALLY_ANSWERED / ESCALATE), the reply text, and a source-citation list referencing at least one passage from the Orders FAQ article.
- Within 1 s of `REPLY_RECORDED`, the card reaches `EVALUATED` and shows a grounding score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a reply with a fabricated passage citation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-customer-query.json` includes a deliberately malformed entry (a `sourcePassageId` that does not exist in the retrieved set).

**Steps:**
1. Submit any seeded query three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser dev tools network panel (`/api/conversations/sse`).

**Expected:**
- The third submission's first agent iteration produces a reply with a fabricated passage id.
- The `before-agent-response` guardrail rejects it. The malformed reply NEVER lands in `ConversationEntity` — there is no `ReplyRecorded` event with the invalid payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a valid reply. The card transitions to `REPLY_RECORDED` with a reply whose every `sourcePassageId` exists in the retrieved set.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming the failed check.

## J3 — Evidence-thin reply receives grounding score 1

**Preconditions:** Mock LLM mode. The mock's evidence-thin entry returns a reply with an empty `sourcePassageIds` list and a minimal `answerText`.

**Steps:**
1. Submit the returns-policy seed query (mock selection is deterministic by seed — see `MockModelProvider.seedFor`).
2. Wait for `EVALUATED`.

**Expected:**
- The verdict lands well-formed (the guardrail asserts at least one passage id is present — but the evidence-thin entry passes the structural check because the mock is crafted to include a valid passage id with no actual support in the text).
- The grounding score chip shows **1** and the rationale reads "Reply cites a passage but answer text does not reflect its content; grounding is nominal."
- The card border highlights red and shows a "Flag for review" label. A support manager reviewing the log can identify this reply for inspection.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a custom query containing the literal strings `jane.doe@example.com`, `ORD-99999`, and `4111-1111-1111-1111` (a test credit-card-like number).
2. Wait for `REPLY_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).
4. Fetch `GET /api/conversations/{id}` and read `query.rawQuery`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-EMAIL]`, `[REDACTED-ORDER-ID]`, `[REDACTED-PCN]`. The raw strings do not appear.
- `query.rawQuery` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `order-id`, `payment-card-number` (at minimum).

## J5 — Escalation flag consistency enforced by guardrail

**Preconditions:** Mock LLM mode. The mock's malformed-escalation entry returns `outcome = ESCALATE` but `escalation = false`.

**Steps:**
1. Submit the account-reset seed query (mock deterministically selects the malformed-escalation entry for this seed on the first iteration of every 3rd conversation).
2. Watch the conversation lifecycle.

**Expected:**
- The first agent iteration returns `outcome = ESCALATE` with `escalation = false` — a consistency violation.
- The guardrail rejects it and logs `escalation-flag-mismatch`.
- The second iteration returns a consistent response. The card transitions to `REPLY_RECORDED` with either a valid `ESCALATE` (escalation=true, escalationReason non-empty) or a non-escalation outcome.
- The UI shows the escalation banner if and only if the final reply has `escalation = true`.
