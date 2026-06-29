# User journeys — ad-creator-pipeline

## J1 — Submit a product URL and get an ad package

**Preconditions:** Service running on declared port (`http://localhost:9438/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded product `Zenith Wireless Headphones` has a matching `src/main/resources/sample-data/products/zenith-wireless-headphones.json` file.

**Steps:**
1. Open `http://localhost:9438/` → App UI tab.
2. From the **Pick a seeded product** dropdown, select `Zenith Wireless Headphones`.
3. Click **Generate ad**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `SCRAPING` within 1 s more.
- Within ~20 s the card reaches `SCRAPED`. The right pane shows the Scraped-profile panel with ≥ 4 attribute rows, a targetAudience value, and at least one brand constraint.
- Within ~20 s more the card reaches `COPIED`. The right pane shows all three format variant sub-panels (INSTAGRAM, FACEBOOK, SEARCH), each with a non-empty headline, body, and call-to-action. Every body references at least one attribute name from the scraped profile.
- Within ~20 s more the card reaches `VISUAL_GENERATED`, then `EVALUATED` within 1 s of that. The right pane shows a `VisualSpec` with a non-empty `imagePrompt` containing the product name and an `aspectRatio` of `1:1`. The eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Brand-safety guardrail blocks a disallowed superlative

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-copy.json` includes one entry whose first response contains "Best wireless headphones guaranteed!" — a disallowed superlative that triggers `BrandSafetyGuardrail`.

**Steps:**
1. Submit the seeded product URL for `Zenith Wireless Headphones` twice in a row (J1 steps × 2).
2. Watch the second submission's lifecycle in the network panel of the browser dev tools (`/api/ad-jobs/sse`).

**Expected:**
- On the second submission's `copyStep`, the agent's first response contains the disallowed phrase. `BrandSafetyGuardrail` rejects the response; a `BrandSafetyRejected{violations: [{rule: "disallowed_superlative", offendingText: "best wireless headphones guaranteed", suggestion: "..."}]}` event lands on the entity.
- The rejected response NEVER reaches `AdJobEntity.recordCopy` — there is no `CopyDrafted` event for the violating text in the entity log.
- The agent's second iteration produces clean copy. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the red dot indicating a brand-safety rejection fired. The rejection-log strip on the right pane shows the one blocked response with its full violation detail.

## J3 — Scraping-policy guardrail blocks an out-of-list URL

**Preconditions:** Service running with the mock LLM selected. The mock's `scrape-product.json` includes one entry whose tool_calls array starts with `fetchProductPage("https://competitor.example.com/rival-headphones")` — a URL outside the allow-list.

**Steps:**
1. Submit any seeded product URL three times in a row.
2. Watch the third submission's lifecycle in the network panel (`/api/ad-jobs/sse`).

**Expected:**
- On the third submission's `scrapeStep`, the agent's first iteration calls `fetchProductPage` with the competitor URL. `ScrapingPolicyGuardrail` rejects the call; a `ScrapingRejected{url: "https://competitor.example.com/...", reason: "scraping-policy-violation: url not on allow-list"}` event lands on the entity.
- The `fetchProductPage` tool body NEVER executes for the disallowed URL — there is no network request in the access log.
- The agent's second iteration uses the approved product URL. The card eventually reaches `EVALUATED` as in J1.
- The card shows the orange dot indicating a scraping-policy rejection fired. The rejection-log strip shows the blocked URL and the reason.

## J4 — Missing attribute grounding flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `draft-copy.json` includes one entry where all three `AdVariant.body` strings are generic ("This product is worth buying.") and reference no attribute names from the scraped `ProductProfile`.

**Steps:**
1. Submit any seeded product URL six times. (The ungrounded-copy entry is selected once in every six runs by the mock's `seedFor(adJobId)` modulo.)
2. Watch the live list until a card's border highlights red.

**Expected:**
- The flagged ad job lands well-formed (both guardrails passed — the copy contains no superlatives or competitor names, and the scrape used the approved URL).
- The eval score chip shows **1** (or 2 if the call-to-action is present) and the rationale reads: *"Attribute grounding failed: no AdVariant.body references any ProductAttribute.name from the ProductProfile."*
- The card's border highlights red. The marketer knows to regenerate or manually revise the copy before publishing.
- The other five ad jobs in the run scored ≥ 4.

## J5 — Custom product URL not in the fixture corpus

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/products/<slug>.json` exists for the user's URL.

**Steps:**
1. In the App UI, paste a custom product URL (e.g. `https://example.com/products/unknown-gadget`) into the product URL field.
2. Click **Generate ad**.

**Expected:**
- `ScrapeTools.fetchProductPage` returns an empty HTML string (no matching fixture file).
- `ScrapeTools.extractAttributes` returns a `ProductProfile` with `attributes = []` and empty `brandConstraints`.
- The workflow advances to `copyStep`. The COPY task returns an `AdCopy` with `primaryHeadline = "(no product attributes found)"` and empty `variants` list (the agent's refusal behaviour per `prompts/ad-creator-agent.md`).
- The pipeline advances to `visualStep`. `VisualTools.buildImagePrompt` produces a minimal prompt referencing only the product URL (no attributes). The workflow advances to `evalStep`.
- The eval score chip shows 1 (attribute grounding failed; variant coverage failed; call-to-action absent; only visual-copy alignment partially satisfied). The rationale names the largest gap.
- The pipeline completes; nothing crashes; the empty ad package is honestly empty.

## J6 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded product URL.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the adJobId.

**Expected:**
- The SCRAPE task's log entries show only `fetchProductPage` and `extractAttributes` calls.
- The COPY task's log entries show only `draftHeadline` and `draftBody` calls.
- The VISUAL task's log entries show only `buildImagePrompt` and `selectAspectRatio` calls.
- No cross-phase calls appear. (If the mock-LLM policy-violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is SCRAPE → COPY → VISUAL. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.
