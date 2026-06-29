# User journeys — currency-agent

## J1 — Submit a USD→EUR conversion and get a result

**Preconditions:** Service running on declared port (`http://localhost:9423/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9423/` → App UI tab.
2. Set **From** to `USD`, **To** to `EUR`, **Amount** to `1500.00`, **Snapshot** to `spot`.
3. Click **Load seeded example** (or fill the fields manually).
4. Click **Convert**.

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `RATE_ATTACHED` within 1 s. The right-pane detail shows the rate snapshot: label, rate, capturedAt timestamp, and source.
- Within 30 s the card reaches `RESULT_RECORDED`. The right pane shows: the converted amount (e.g. `1384.6500 EUR`), the rate applied, a non-empty `confidenceNote`, and a 1–2-sentence `marketContext`.
- Within 1 s of `RESULT_RECORDED`, the card reaches `EVALUATED` and shows a freshness score chip (1–5) and a one-line rationale.

## J2 — Guardrail blocks a malformed result

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `convert-currency.json` includes deliberately malformed entries (one with null `convertedAmount`; one with a blank `fromCurrency`).

**Steps:**
1. Submit three conversions in a row using the seeded USD→EUR spot example.
2. Watch the third submission's lifecycle in the browser network panel (`/api/conversions/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed result.
- The `before-agent-response` guardrail rejects it. The malformed result NEVER lands in `ConversionEntity` — there is no `ResultRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed result. The card transitions to `RESULT_RECORDED` with a valid `ConversionResult`.
- The service log shows one `guardrail.reject` line per rejected iteration, naming which check failed.

## J3 — Stale rate snapshot scores low on freshness

**Preconditions:** Service running. The seeded rate-snapshots file includes an `end-of-day` snapshot timestamped 30 h ago (seed is designed to exercise this band).

**Steps:**
1. Set **Snapshot** dropdown to `end-of-day`.
2. Submit any currency pair (e.g. `GBP` → `JPY`).
3. Wait for `EVALUATED`.

**Expected:**
- The conversion reaches `EVALUATED` with a freshness eval score of **1** or **2**.
- The rationale reads something like "Rate snapshot is 30 hours old; confidence is low."
- The card's border highlights amber. The freshness score chip is amber-coloured.

## J4 — Three conversions in parallel are independent

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Without waiting for each to finish, submit three conversions in rapid succession:
   - USD → EUR, amount 100, snapshot `spot`
   - EUR → GBP, amount 500, snapshot `morning-fix`
   - USD → JPY, amount 1000, snapshot `end-of-day`

**Expected:**
- All three cards appear in the live list, each with a unique `conversionId`.
- Each card progresses through its own lifecycle independently. No card's result appears on another card.
- All three reach `EVALUATED` within 60 s of each other.
- Each result reflects the correct currency pair and rate from the matching snapshot label.
