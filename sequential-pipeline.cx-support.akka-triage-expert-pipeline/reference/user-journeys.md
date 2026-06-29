# User journeys — triage-expert-multi-agent-workflow

## J1 — Submit a billing dispute and receive a grounded recommendation

**Preconditions:** Service running on declared port (`http://localhost:9892/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded category `Billing dispute` has a matching entry in `src/main/resources/sample-events/issues.jsonl` and a paired account record in `src/main/resources/sample-data/accounts/`.

**Steps:**
1. Open `http://localhost:9892/` → App UI tab.
2. From the **Pick a seeded issue** dropdown, select `Billing dispute`.
3. Click **Submit issue**.

**Expected:**
- The new card appears with status `CREATED` within 1 s, then `GATHERING` within 1 s more.
- Within ~20 s the card reaches `GATHERED`. The right pane shows the Customer info panel with non-empty name, email, account ID, phone, category = `BILLING_DISPUTE`, and a derived urgency level.
- Within ~20 s more the card reaches `SUMMARIZED`. The Triage summary panel shows a 1–2 sentence problem statement and 2–5 key facts.
- Within ~2 s the card reaches `SANITIZED`. The Sanitized summary panel shows the problem statement with at least one PII placeholder (`[CUSTOMER_NAME]`, `[EMAIL]`, or `[ACCOUNT_ID]`); the PII scrub audit strip lists the redacted token types.
- Within ~30 s more the card reaches `RECOMMENDED`, then `EVALUATED` within 1 s. The Recommendation panel shows guidance text, ≥ 1 cited article with a valid article ID, and a `Requires escalation` badge if applicable. The eval score chip shows ≥ 3/5.
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Guardrail blocks a non-compliant recommendation and the agent self-corrects

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `compose-recommendation.json` includes one entry whose `guidanceText` contains `"will fix your issue"` — this is the deliberately blocked entry.

**Steps:**
1. Submit any seeded issue three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser network panel (`/api/cases/sse`).

**Expected:**
- On the third submission's `recommendStep`, the agent's first iteration produces a recommendation containing `"will fix your issue"`. `RecommendationGuardrail` blocks it; a `GuardrailBlocked{rule: "no-diagnostic-promise", excerpt: "will fix your issue"}` event lands on the entity.
- The blocked recommendation is NEVER written onto the entity — there is no `RecommendationComposed` event preceding the block in the audit log.
- The agent's second iteration produces a compliant recommendation. The case reaches `EVALUATED` as in J1.
- The card shows the small red dot indicating a block fired. The guardrail-block log strip on the right pane shows the one blocked response with its rule and excerpt.

## J3 — Sanitized summary contains no raw PII

**Preconditions:** Service running. Any model provider. The seeded account for the `Account access locked` category has a customer name, email, account ID, and phone number in `src/main/resources/sample-data/accounts/`.

**Steps:**
1. Submit the seeded issue `Account access locked`.
2. Wait for `SANITIZED`.
3. Inspect `SanitizedSummary.problemStatement` and `SanitizedSummary.keyFacts` in the right pane.

**Expected:**
- Neither the customer's actual name, nor email address, nor account ID, nor phone number appears as plain text in `problemStatement` or any `keyFacts` item.
- The `redactedTokens` list includes at least `CUSTOMER_NAME` and `ACCOUNT_ID`.
- The `IssueSummary.problemStatement` visible in the Triage summary panel (pre-sanitization) DOES contain the customer's name, confirming the sanitizer had material to scrub.
- The PII scrub audit strip on the Sanitized summary panel lists the scrubbed token types.

## J4 — Recommendation citing an unknown article scores 1

**Preconditions:** Mock LLM mode. The mock's `compose-recommendation.json` includes one entry whose `citedArticles` list contains an `articleId` not present in `src/main/resources/knowledge-base/articles/`.

**Steps:**
1. Submit any seeded issue six times. (The uncited-article entry is selected once in every six runs by the mock's `seedFor(caseId)` modulo.)
2. Watch the live list until a case card border highlights red.

**Expected:**
- The flagged case lands with a well-formed `Recommendation` (the guardrail checks content policy, not article provenance).
- The eval score chip shows **1** and the rationale reads: *"Article provenance failed: articleId 'KB-XXXX' cited in recommendation does not exist in knowledge base."*
- The card's border highlights red. The support agent sees the flag before contacting the customer.
- The other five cases in the run scored ≥ 3.

## J5 — Category mismatch between triage and recommendation scores low

**Preconditions:** Service running. Any model provider that can be prompted or mocked to return a `Recommendation` with `category = ACCOUNT_ACCESS` for a case whose `SanitizedSummary.category = BILLING_DISPUTE`.

**Steps:**
1. Submit the seeded issue `Billing dispute`.
2. Wait for `EVALUATED`.
3. Check the eval score chip.

**Expected:**
- If `recommendation.category != sanitizedSummary.category`, the category-alignment check fails and the score is at most 3 (base 1 + article citation + article provenance, but no category alignment point).
- The rationale reads: *"Category alignment failed: recommendation.category 'ACCOUNT_ACCESS' does not match sanitizedSummary.category 'BILLING_DISPUTE'."*
- The pipeline does not crash; the mismatched recommendation is committed and flagged.

## J6 — Issue description with no matching account produces a graceful fallback

**Preconditions:** Service running. Any model provider. No account record exists for the submitted account ID.

**Steps:**
1. Type a custom issue description into the issue field that references an account ID not in the sample-data corpus (e.g., `"My account ACC-99999 is locked and I need access."`).
2. Click **Submit issue**.

**Expected:**
- `IntakeTools.lookupCustomerAccount("ACC-99999")` returns a fallback `AccountRecord` with `name = "[unknown]"`, `email = "[unknown]"`, `phone = "[unknown]"`.
- `TriageAgent` produces a valid `CustomerInfo` with the fallback values and derives urgency from the description text.
- `PiiSanitizer` finds no literal name, email, or phone to replace; `redactedTokens` is empty or contains only `ACCOUNT_ID`.
- The pipeline completes to `EVALUATED`. The support agent sees the fallback markers and knows manual account lookup is needed.
- Nothing crashes; the partial customer info is honestly partial.
