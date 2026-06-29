# User journeys — product-catalog-ad-generation

## J1 — Submit a product and get a scored ad

**Preconditions:** Service running on declared port (`http://localhost:9343/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded product `Wireless Noise-Cancelling Headphones` has a matching `src/main/resources/sample-data/catalog/wireless-noise-cancelling-headphones.json` file.

**Steps:**
1. Open `http://localhost:9343/` → App UI tab.
2. From the **Pick a seeded product** dropdown, pick `Wireless Noise-Cancelling Headphones`.
3. Click **Generate Ad**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `ENRICHING` within 1 s more.
- Within ~20 s the card reaches `ENRICHED`. The right pane shows the Enriched-product table with non-empty category, pricingTier, at least 2 keyFeatures, and a targetAudience.
- Within ~20 s more the card reaches `DRAFTED`. The right pane shows an `AdDraft` with a non-empty headline, a URL-safe callToAction, and at least 2 AdPlacement entries each with a charCount within its declared limit.
- Within ~20 s more the card reaches `APPROVED`, then `SCORED` within 1 s of that. The right pane shows a `ProductAd` and a compliance score chip ≥ 4.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Brand guardrail blocks a policy-violating draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-ad.json` includes one entry whose headline contains a prohibited word.

**Steps:**
1. Submit any seeded product three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/ad-jobs/sse`).

**Expected:**
- On the third submission's `draftStep`, the agent's first DRAFT-phase response contains a prohibited word in the headline. `BrandGuardrail` rejects it; a `GuardrailRejected{phase: "DRAFT", field: "headline", rule: "prohibited-word", found: "cheapest"}` event lands on the entity.
- The rejected draft is NOT written as `AdDrafted` — there is no `AdDraft` record on the entity until the agent's compliant retry lands.
- The agent's second iteration returns a headline free of prohibited words. The card eventually reaches `SCORED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected response with its full structured violation list.

## J3 — Missing brand element flags compliance score 2

**Preconditions:** Mock LLM mode. The mock's `review-ad.json` includes one entry whose headline lacks any approved brand element from `elements.json`.

**Steps:**
1. Submit any seeded product six times. (The no-brand-element entry is selected once in every six runs by the mock's `seedFor(jobId)` modulo.)
2. Watch the live list until a job's card border highlights red.

**Expected:**
- The flagged job lands with a `ProductAd` that passed all other brand rules (no prohibited words, valid CTA slug, placement parity satisfied) but whose headline contains no brand element.
- The compliance score chip shows **2** and the rationale reads: *"Headline element check failed: no approved brand element found in headline."*
- The card's border highlights red. The reviewer knows to check the headline before placement.
- The other five jobs in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded product.
2. Wait for `SCORED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the jobId.

**Expected:**
- The ENRICH task's log entries show only `lookupCategory` and `inferAttributes` calls.
- The DRAFT task's log entries show only `composeHeadline` and `composeCopy` calls.
- The REVIEW task's log entries show only `checkBrandElements` and `finaliseAd` calls.
- No cross-phase tool calls appear.
- The order of tasks in the log is ENRICH → DRAFT → REVIEW. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Placement character-limit violation caught at REVIEW

**Preconditions:** Mock LLM mode. Any seeded product with a `display` placement entry that exceeds 300 characters in the draft.

**Steps:**
1. Submit any seeded product.
2. Watch for a `GuardrailRejected` event in the SSE stream tagged with `rule: "placement-char-limit"`.

**Expected:**
- The `BrandGuardrail` intercepts the REVIEW-phase response and finds a `display` placement with `charCount > 300`.
- A `GuardrailRejected{phase: "REVIEW", field: "placements[display]", rule: "placement-char-limit", found: "312"}` event lands on the entity.
- The agent trims the display copy and re-submits; the revised `charCount` is within limit.
- The final `ProductAd` has a `display` placement with `charCount ≤ 300`. The compliance score reflects all rules passing.

## J6 — Custom product name with no matching catalog entry

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/catalog/<slug>.json` exists for the user's product.

**Steps:**
1. In the App UI, type a custom product name (e.g. `Artisanal Wooden Spoon Set`) into the product name field.
2. Click **Generate Ad**.

**Expected:**
- `EnrichTools.lookupCategory` returns `"uncategorized"`.
- `EnrichTools.inferAttributes` returns a `ProductAttributes` with empty `keyFeatures` and `targetAudience = "general"`.
- The agent's DRAFT_AD task receives an `EnrichedProduct` with insufficient attributes and returns an `AdDraft` with `headline = "(insufficient product data)"` and an empty `placements` list.
- The workflow advances to `reviewStep`. The REVIEW_AD task returns a `ProductAd` with no placements.
- The compliance score is 1 (headline element missing, no placements so parity fails). The rationale names the gaps.
- The pipeline completes; nothing crashes; the empty ad is honestly empty.
