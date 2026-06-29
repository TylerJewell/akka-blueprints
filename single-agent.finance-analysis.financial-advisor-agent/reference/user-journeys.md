# User journeys — financial-advisor-agent

## J1 — Submit an IRA client profile and receive a rebalancing advisory

**Preconditions:** Service running on declared port (`http://localhost:9699/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9699/` → App UI tab.
2. From the **Client profile** dropdown, pick `IRA conservative (5 holdings)`.
3. Click **Load seeded example** to fill the profile and question textarea.
4. Click **Ask advisor**.

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted profile JSON; the identifier-category chips show `account-id` (at minimum — depending on the seed profile).
- Within 30 s the card reaches `RESPONSE_RECORDED`. The right pane shows: a risk rating badge (CONSERVATIVE), the recommendation paragraph, and a holding-advice row for *each of the 5 holdings*. Every row has a non-empty `rationale`, a `suggestedWeight` in `[0.0, 1.0]`, and an `assetClass` within the IRA-authorized set.
- Within 1 s of `RESPONSE_RECORDED`, the card reaches `AUDITED` and shows the audit timestamp and digest chip.

## J2 — Guardrail blocks an unauthorized asset-class recommendation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `advise-client.json` includes a deliberately non-conforming entry recommending a crypto instrument for an IRA client.

**Steps:**
1. Submit the IRA conservative seeded client three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/advisories/sse`).

**Expected:**
- The third submission's first agent iteration produces an advisory containing a crypto holding.
- The `before-agent-response` guardrail rejects it (asset-class check fails for the IRA account type). The non-conforming response NEVER lands in `AdvisoryEntity` — there is no `ResponseRecorded` event with the prohibited holding.
- The agent loop retries on iteration 2 and produces a compliant advisory. The card transitions to `RESPONSE_RECORDED` with a response that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line for the rejected iteration, naming `unauthorized-asset-class` as the failed check.

## J3 — Regulated identifiers never reach the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a client profile containing the literal strings `SSN: 987-65-4321`, `Account: BR-00012345`, and `EIN: 12-3456789` embedded in a custom profile JSON.
2. Wait for `RESPONSE_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/advisories/{id}` and read `request.profile` from the JSON.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-SSN]`, `[REDACTED-ACCOUNT]`, `[REDACTED-EIN]`. The raw strings do not appear.
- `request.profile` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.identifierCategoriesFound` lists `ssn`, `account-id`, `ein` (at minimum).

## J4 — Per-holding completeness (brokerage 8-holding client)

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the `Brokerage moderate (8 holdings)` seeded client with the diversification question.
2. Wait for `AUDITED`.

**Expected:**
- The advisory's `holdingAdvice` array contains exactly 8 entries, one per holding in the submitted profile. No holdings are silently omitted.
- Each `holdingAdvice[i].ticker` matches one of the 8 tickers in the submitted profile, one-to-one.
- Every `holdingAdvice[i].suggestedWeight` is in `[0.0, 1.0]`.
- If the agent's first iteration omitted a holding, the guardrail would have rejected the response for incomplete coverage — the absence of a `guardrail.reject` in the log confirms the first iteration was complete.

## J5 — Out-of-range weight blocked by guardrail

**Preconditions:** Mock LLM mode. The mock's `advise-client.json` includes an entry where one holding has `suggestedWeight = 1.5`.

**Steps:**
1. Submit any seeded client such that the mock picks the out-of-range entry on the first iteration (see seedFor logic — every 3rd advisory modulo seed).
2. Watch the advisory lifecycle in the SSE stream.

**Expected:**
- The first iteration's response is rejected by the guardrail (weight-bounds check fails).
- The rejection log line names `weight-out-of-bounds` as the failed check.
- The second iteration produces a well-formed response with all weights in `[0.0, 1.0]`. The UI displays only the conforming response.

## J6 — ROTH_IRA client receives only authorized asset classes

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the `ROTH_IRA aggressive (6 holdings)` seeded client with a growth-tilt question.
2. Wait for `AUDITED`.

**Expected:**
- Every `holdingAdvice[i].assetClass` in the response is one of `{equities, fixed-income, ETF}` — the ROTH_IRA-authorized set.
- No entry contains `REITs`, `commodities-ETF`, or any alternative asset.
- If the agent had suggested a REIT or commodity ETF, the guardrail would have blocked it; the absence of a `guardrail.reject` in the log confirms the response was immediately compliant.
