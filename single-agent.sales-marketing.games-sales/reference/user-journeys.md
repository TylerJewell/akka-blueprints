# User journeys — games-sales

## J1 — Submit a PS5 action query and get a recommendation

**Preconditions:** Service running on `http://localhost:9801/`; a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9801/` → App UI tab.
2. From the **Load seeded query** dropdown, pick `PS5 co-op action under $70`.
3. Confirm the **Platform**, **Genre**, and **Max budget** fields are populated from the seeded query.
4. Click **Get recommendations**.

**Expected:**
- The new session card appears in the live list with status `QUERYING` within 1 s.
- Within 30 s the card transitions to `RECOMMENDED`. The right pane shows: 1–5 ranked game suggestion cards, each with a title, platform label, formatted price, a confidence score bar, a rationale sentence, and an optional upsell chip.
- Every `catalogId` in the displayed suggestions matches an entry in `game-catalog.jsonl` — no unknown titles appear.
- All confidence score bars are non-zero and within the 0–100% render range.

## J2 — Guardrail blocks a malformed recommendation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `recommend-games.json` includes a deliberately malformed entry with `catalogId = "fake-game-xyz"` and one with `confidenceScore = 1.5`.

**Steps:**
1. Submit any seeded query three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/sessions/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed recommendation.
- The `before-agent-response` guardrail rejects it. The malformed recommendation NEVER lands in `SessionEntity` — there is no `RecommendationRecorded` event with the hallucinated `catalogId` or out-of-range confidence score.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed recommendation. The card transitions to `RECOMMENDED` with a recommendation that passes all five guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed (`unknown-catalog-id` or `confidence-out-of-range`).

## J3 — Follow-up query refines an existing recommendation

**Preconditions:** Service running. A session has reached `RECOMMENDED` status (J1 completed).

**Steps:**
1. In the right pane of the selected session, locate the **Refine your request** textarea.
2. Type "Actually, I'd prefer something with a strong single-player story mode" and click **Send follow-up**.

**Expected:**
- The session card transitions to `FOLLOW_UP_QUERYING` within 1 s.
- Within 30 s the card transitions to `FOLLOW_UP_RECOMMENDED`. The right pane shows a **Follow-up** recommendation section below the original recommendation.
- The follow-up suggestions differ from the original top-ranked title (the agent received the prior recommendation as an attachment and avoided straight repetition).
- The follow-up summary paragraph begins with a phrase acknowledging the prior result (e.g., "Building on the earlier suggestions...").

## J4 — Guardrail exhaustion causes session failure

**Preconditions:** Mock LLM mode. A custom mock entry is configured to return `"fake-game-xyz"` on all three iterations (simulating a persistent hallucination).

**Steps:**
1. Trigger the three-iteration-fail mock entry (e.g., by adjusting the session seed so all three iterations pick the malformed entry).
2. Submit a query and wait.

**Expected:**
- The guardrail rejects all three iterations.
- `SessionWorkflow`'s `queryStep` exhausts its retry budget and transitions to the `error` step.
- `SessionEntity` receives a `fail` command and transitions to `FAILED`.
- The session card in the UI shows status `FAILED` (red pill).
- The service log shows three consecutive `guardrail.reject` lines followed by one `workflow.step.failed` line.
- The prior session data (preferences, `createdAt`) is preserved on the entity for audit.

## J5 — Budget constraint is respected in suggestions

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a query with `maxBudgetCents = 2999` (i.e., $29.99 limit) and `platform = PC`.
2. Wait for `RECOMMENDED`.

**Expected:**
- Every suggestion with `confidenceScore ≥ 0.7` has `priceCents ≤ 2999`.
- If the agent includes a game above the budget, its `confidenceScore` is below 0.7 and its `rationale` explicitly mentions the price difference.
- No suggestion has a `priceCents` value that exceeds the budget by more than 50% without a rationale note.
