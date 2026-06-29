# User journeys — support-next-steps

## J1 — Submit a billing ticket and get a recommendation

**Preconditions:** Service running on declared port (`http://localhost:9220/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9220/` → App UI tab.
2. From the **Product area** dropdown, pick `Billing`.
3. Click **Load seeded example** to fill the subject and ticket body textarea.
4. Click **Submit ticket**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted ticket body; the PII category chips show `email`, `account-number` (at minimum — depending on the seed content).
- Within 30 s the card reaches `RECOMMENDATION_RECORDED`. The right pane shows: a confidence badge (HIGH / MEDIUM / LOW), the rationale paragraph, and at least two step rows, each with a non-empty description, an action type chip, a confidence level, and a `resolutionRef` value.
- Within 1 s of `RECOMMENDATION_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Ungrounded recommendation flags eval score 1

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `advise-ticket.json` includes a deliberate "ungrounded" entry where all steps have empty `resolutionRef` strings.

**Steps:**
1. Submit four tickets in a row using the Billing seed (J1 steps × 4). The mock selects the ungrounded entry on every 4th ticket.
2. Watch the fourth card's lifecycle in the live list.

**Expected:**
- The fourth card's recommendation lands with `overallConfidence: LOW` and all steps showing empty resolution references.
- The eval score chip shows **1** and the rationale reads that steps lack grounding citations.
- The card border highlights red. The support agent knows to review this recommendation before acting.
- The first three cards are unaffected — they score 3 or higher.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a ticket containing the literal strings `jane.doe@acme.com`, `phone: 555-867-5309`, and `customer ID: CUST-99182`.
2. Wait for `RECOMMENDATION_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/tickets/{id}` and read `request.ticketBody`.

**Expected:**
- The logged LLM call body (in the `ticket.txt` attachment) contains only the redacted forms: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-ACCOUNT]`. The raw strings do not appear.
- `request.ticketBody` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `phone`, `account-number` (at minimum).

## J4 — Late-joining SSE client receives full state

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a ticket and wait for it to reach `EVALUATED`.
2. Open a new browser tab and connect to `GET /api/tickets/sse`.

**Expected:**
- The SSE stream immediately emits the current state of all tickets, including the `EVALUATED` ticket submitted in step 1.
- The late-joining client does not see a blank or partial state; it receives the full `Ticket` row including recommendation and eval fields.
- Subsequent state transitions from new ticket submissions arrive in real time on the same stream.

## J5 — Authentication ticket resolution path

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Product area** dropdown, pick `Authentication`.
2. Click **Load seeded example** to fill the authentication seed ticket (a login-flow error).
3. Click **Submit ticket** and wait for `EVALUATED`.

**Expected:**
- The verdict's `steps` list contains at least one step with `actionType: VERIFY` and a non-empty `resolutionRef` from the authentication resolution library.
- The overall confidence is not `LOW` — the authentication library has at least one matching past case.
- The eval score chip is 3 or higher, confirming at least one step is grounded and actionable.
