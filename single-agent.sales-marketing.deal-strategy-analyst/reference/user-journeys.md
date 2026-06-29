# User journeys — deal-strategy-analyst

## J1 — Submit an enterprise-new deal and get a recommendation

**Preconditions:** Service running on declared port (`http://localhost:9706/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9706/` → App UI tab.
2. From the **Deal type** dropdown, pick `enterprise-new`.
3. Click **Load seeded example** to fill the deal name, context textarea, and stakeholder rows.
4. Click **Analyse deal**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted context with PII category chips above (`email`, `phone` at minimum).
- Within 30 s the card reaches `RECOMMENDATION_RECORDED`. The right pane shows: an urgency badge, the executive-summary paragraph, and a ranked next-step table with at least 3 rows. Every row has a non-empty `stakeholderRole`, `rationale`, `suggestedDeadline`, and a valid `actionType`.
- Within 1 s of `RECOMMENDATION_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed recommendation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `analyse-deal.json` includes deliberately malformed entries (a next step whose `stakeholderRole` is not in the submitted stakeholder list; a next step with an `actionType` value outside the enum).

**Steps:**
1. Submit any seeded deal three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/deals/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed recommendation.
- The `before-agent-response` guardrail rejects it. The malformed recommendation NEVER lands in `DealEntity` — there is no `RecommendationRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed recommendation. The card transitions to `RECOMMENDATION_RECORDED` with a recommendation that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration naming which check failed.

## J3 — Actionability-thin recommendation flags eval score 1

**Preconditions:** Mock LLM mode. A rep submits a deal whose mock entry returns next steps with empty `suggestedDeadline` fields and single-word `rationale` strings.

**Steps:**
1. Submit the renewal-risk seeded deal using the `custom` deal type entry that maps to the "deadline-empty" mock-response entry (see `src/main/resources/mock-responses/analyse-deal.json`).

**Expected:**
- The recommendation lands well-formed (the guardrail checks structural validity, not content quality).
- The eval score chip shows **1** and the rationale reads "Next steps have no suggested deadlines and rationale strings are too brief to be actionable."
- The card's border highlights red. The rep knows to treat this recommendation with caution before acting.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged.

**Steps:**
1. Submit a deal context containing the literal strings `sarah.johnson@buyer.com`, `+1-555-867-5309`, and `ACME CORPORATION` (in all-caps to trigger the confidential-name pattern).
2. Wait for `RECOMMENDATION_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/deals/{id}` and read `context.rawContext`.

**Expected:**
- The logged LLM call body contains redacted tokens only: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-COMPANY]`. The raw strings do not appear.
- `context.rawContext` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `phone`, and at least one confidential-identifier category.

## J5 — Stakeholder completeness

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the competitive-selection seeded deal, which has 5 stakeholders.
2. Wait for `EVALUATED`.

**Expected:**
- Every `nextSteps[].stakeholderRole` in the recommendation matches a role from the submitted 5-stakeholder list.
- No invented roles appear in the next-step table.
- If the agent's first iteration included an invented role, the guardrail would have rejected it — absence of any rejection in the log confirms the first iteration was clean.

## J6 — FAILED deal preserves partial state

**Preconditions:** Mock LLM mode. The mock is configured to return malformed responses on all 3 iterations for a specific deal seed (see `analyse-deal.json` "always-malformed" entry).

**Steps:**
1. Submit the deal that maps to the always-malformed mock entry.
2. Wait until the card transitions to `FAILED`.

**Expected:**
- The card status pill shows `FAILED` in red.
- The right pane shows the sanitized context preview (populated before the agent was called) and a failure reason message.
- The `SUBMITTED` and `SANITIZED` events are visible in the entity's event log; `RecommendationRecorded` is absent.
- The rep can resubmit by clicking **Analyse deal** on a new submission form with the same context.
