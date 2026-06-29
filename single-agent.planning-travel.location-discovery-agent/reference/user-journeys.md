# User journeys — location-discovery-agent

## J1 — Submit a coffee-shop search and get a ranked recommendation

**Preconditions:** Service running on declared port (`http://localhost:9150/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9150/` → App UI tab.
2. Click **Downtown SF** to fill the coordinate fields.
3. Set radius to `1 km`, category to `Food & Drink`.
4. Type `coffee shops with good wifi for remote work` in the search query field.
5. Click **Find places**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `COORDINATE_SANITIZED` within 1 s. The right-pane detail shows the bounding-box token (e.g., `bbox:37.77-37.79:-122.42--122.40`) and a count of candidate places within the 1 km radius.
- Within 30 s the card reaches `RECOMMENDATION_RECORDED`. The right pane shows: a ranked list of at least 3 places, each with a relevance score, distance badge, and a non-generic one-sentence rationale.
- Within 1 s of `RECOMMENDATION_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed recommendation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `discover-places.json` includes deliberately malformed entries (a result with a `placeId` not in the candidate list; a result with a `relevanceScore` of 0).

**Steps:**
1. Submit any seeded area search three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/searches/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed recommendation.
- The `before-agent-response` guardrail rejects it. The malformed recommendation NEVER lands in `SearchEntity` — there is no `RecommendationRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed recommendation. The card transitions to `RECOMMENDATION_RECORDED` with a recommendation that satisfies all three guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Generic-rationale recommendation flags eval score 1

**Preconditions:** Mock LLM mode. The mock response file contains an entry whose `rationale` strings are all generic filler (e.g., "great place", "highly recommended", "worth visiting").

**Steps:**
1. Submit the Midtown NYC seeded area with query `bookstores and independent shops`.
2. The mock is configured to return the generic-rationale entry for this search seed.
3. Wait for `EVALUATED`.

**Expected:**
- The recommendation lands well-formed (the guardrail only checks structural validity, not rationale quality).
- The eval score chip shows **1** and the rationale reads "Rationale strings are generic filler; results are not anchored to the specific query."
- The card's border highlights red. The user knows to treat these results with caution.

## J4 — Exact coordinates never reach the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a search with precise coordinates: latitude `51.5074`, longitude `-0.1278` (central London).
2. Wait for `RECOMMENDATION_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/searches/{id}` and read `request.rawLatitude` and `request.rawLongitude`.

**Expected:**
- The logged LLM call body contains the bounding-box token only (e.g., `bbox:51.50-51.52:-0.13--0.11`). The literal values `51.5074` and `-0.1278` do not appear.
- `request.rawLatitude` and `request.rawLongitude` in the JSON still contain the precise values — the audit log preserves them.
- `sanitized.boundingBoxToken` shows the coarsened token.

## J5 — Radius filtering limits candidates correctly

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the Downtown SF area with radius `500 m`.
2. Wait for `COORDINATE_SANITIZED`. Note the candidate place count shown in the right pane.
3. Submit the same Downtown SF area with radius `2 km`.
4. Wait for `COORDINATE_SANITIZED`.

**Expected:**
- The 500 m search shows fewer candidates than the 2 km search.
- The agent's recommendation for the 2 km search may include places not present in the 500 m candidate list — the guardrail accepts both because the place ids are drawn from the respective candidate sets.
- No recommendation references a place id that was outside the radius for that search.

## J6 — Category filter excludes non-matching places

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the Central London area with category filter `Arts & Entertainment` and query `museums and galleries`.
2. Wait for `RECOMMENDATION_RECORDED`.

**Expected:**
- Every place in the `places` list has `category` equal to one of the Arts & Entertainment entries in the seeded dataset. No Food & Drink or Retail places appear.
- The agent's summary acknowledges the category constraint.
- If there are fewer than 3 Arts & Entertainment candidates within the radius, the recommendation contains only the available matches — no hallucinated venues fill the gap.
