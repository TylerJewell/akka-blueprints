# User journeys — games-sales-assistant

## J1 — Browse by genre and price

**Preconditions:** Service running on declared port (`http://localhost:9306/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded `catalog.jsonl` contains at least 3 action-RPG titles priced under $40.

**Steps:**
1. Open `http://localhost:9306/` → App UI tab.
2. Click **New session** and enter shopper id `shopper-demo-1`.
3. In the Ask textarea, type: "What action RPGs do you have under $40 on PC?"
4. Click **Ask**.

**Expected:**
- The turn appears in the conversation view with status chip `PENDING` within 1 s.
- Within 20 s, the chip transitions to `ANSWERED`. The agent's answer paragraph states how many matching titles are available.
- The recommendation cards block shows 1–5 `GameTitle` cards. Every card's `titleId` exists in `catalog.jsonl`. Every card's `platform` is `PC`. Every card's `priceUsd` is less than 40.
- No card shows a title marked `inStock: false` as the primary recommendation.

## J2 — Guardrail blocks an off-catalog title

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The `assist-shopper.json` mock file contains a deliberately malformed entry with a `Recommendation.titleId` that is not present in `catalog.jsonl`.

**Steps:**
1. Click **New session** with shopper id `shopper-demo-2`.
2. Submit any question that triggers the mock to select its malformed entry on the first iteration (every 3rd session by the seed logic — use the 3rd session created in this run).
3. Watch the turn in the network panel of the browser dev tools (`/api/sessions/sse`).

**Expected:**
- The first agent iteration produces a response with an off-catalog `titleId`.
- `ResponseGuardrail` rejects it. The rejected response NEVER lands in `SessionEntity` — there is no `TurnAnswered` event with the malformed payload.
- The agent loop retries on iteration 2 (or 3 if needed) and produces a response whose every `Recommendation.titleId` exists in `CatalogIndex`. The chip transitions to `ANSWERED` with a valid recommendation set.
- The service log shows one `guardrail.reject` line per rejected iteration with `check: catalog-title-not-found` and the offending `titleId`.

## J3 — Order history retrieval

**Preconditions:** Service running. The seeded `orders.jsonl` contains at least 2 `OrderLine` entries for `shopper-demo-3`.

**Steps:**
1. Click **New session** with shopper id `shopper-demo-3`.
2. Submit the question: "What have I purchased recently?"
3. Wait for `ANSWERED`.

**Expected:**
- The agent's answer paragraph mentions the number of past orders found.
- The orders block in the right pane lists the correct `OrderLine` entries from the seeded data for `shopper-demo-3`. Each row shows order id, title name, price paid, and purchase date.
- The recommendations block is empty (the question was not a browsing request).

## J4 — Forbidden-topic guardrail blocks a warranty claim

**Preconditions:** Service running with the mock LLM. The `assist-shopper.json` mock file contains an entry whose `answer` includes the phrase "we guarantee this title for 12 months" — a string in the `FORBIDDEN_TOPICS` set.

**Steps:**
1. Click **New session** with shopper id `shopper-demo-4`.
2. Submit a question that triggers the mock's forbidden-topic malformed entry.
3. Observe the session card and service log.

**Expected:**
- The first agent iteration produces an answer containing the warranty phrase.
- `ResponseGuardrail` rejects it on the `forbidden-topics` check. The rejected response does not land in `SessionEntity`.
- If the agent exhausts all 3 iterations with forbidden-topic answers, the turn's chip transitions to `GUARDRAIL_REJECTED` (orange). The session remains `ACTIVE` — the shopper can ask a new question.
- The service log shows `guardrail.reject` with `check: forbidden-topic-in-answer` and the offending phrase.

## J5 — Multi-turn conversation context

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Click **New session** with shopper id `shopper-demo-5`.
2. Submit: "What strategy games do you have on Switch?"
3. Wait for `ANSWERED`. Note the titles returned.
4. In the same session, submit: "Which of those is cheapest?"
5. Wait for `ANSWERED`.

**Expected:**
- The second turn's agent response references one of the titles from the first turn's recommendation set (the conversation context is threaded via `previousTurnIds` in `ShopperContext`).
- The `recommendations` list in the second response contains the title identified as cheapest from the first turn's set. The `priceUsd` is consistent with the catalog value.
- Both turns appear in the conversation view; the second turn's answer text makes sense relative to the first.
