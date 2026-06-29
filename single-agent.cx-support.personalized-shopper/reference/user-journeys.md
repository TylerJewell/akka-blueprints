# User journeys — personalized-shopper

Acceptance journeys. Each defines a precondition, a step sequence, and the expected outcome. The generated system must satisfy all four.

---

## J1 — Electronics shopper receives matched recommendations

**Precondition:** Service is running. The seeded electronics catalog (`catalog.jsonl`) contains at least 5 in-stock products in the `electronics` category with prices in the $50–$300 range and at least one `TechCore` brand product.

**Steps:**
1. Open the App UI tab.
2. Select the "Budget-conscious electronics buyer" seeded profile: preferred categories `[electronics]`, preferred brands `[TechCore]`, price range $50–$200, excluded categories `[apparel]`, no free-text note.
3. Select the `electronics` catalog snapshot.
4. Click **Get recommendations**.
5. Observe the session card appear in `STARTED` state.
6. Wait up to 5 s — the card transitions to `PROFILE_SANITIZED`. The PII stripped list shows `email`, `phone`, `loyalty-card`, `person-name`.
7. Wait up to 30 s — the card transitions through `RECOMMENDING` to `RECOMMENDATIONS_READY`. The outcome badge reads `MATCHED`. At least 3 `ProductRecommendation` entries appear, ranked by relevance. Each entry shows a productId present in the catalog, a price in $50–$200, and a rationale paragraph.
8. Within 2 s — the card transitions to `FRESHNESS_SCORED`. A freshness score chip (1–5) appears. The score rationale references in-stock and current-season counts.

**Expected:** `outcome = MATCHED`, at least 3 ranked recommendations, freshness score visible, no session in `FAILED` state.

---

## J2 — PII never reaches the LLM call log

**Precondition:** Service is running. LLM call logging is enabled (dev mode default).

**Steps:**
1. Submit a custom profile with `shopperEmail = "alex.morgan@example.com"`, `shopperPhone = "+1-555-0192"`, `loyaltyCardNumber = "LC-77340012"`, `shopperName = "Alex Morgan"`.
2. Include a `freeTextNote` of `"Contact me at alex.morgan@example.com for questions."`.
3. Wait for the session to reach `FRESHNESS_SCORED`.
4. Inspect the LLM call log (available in dev-mode console output or via the entity's audit trail).

**Expected:** The strings `alex.morgan@example.com`, `+1-555-0192`, `LC-77340012`, and `Alex Morgan` do not appear anywhere in the LLM call input. The `profile.txt` attachment seen by the agent contains `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-LOYALTY-CARD]`, and `[REDACTED-NAME]` in place of the original values. The entity's `profile.*` fields retain the originals for audit when fetched via `GET /api/sessions/{id}`.

---

## J3 — No-match profile returns a clear explanation

**Precondition:** Service is running. The seeded catalog contains no products in the category `vintage-collectibles`.

**Steps:**
1. Submit a custom profile with `preferredCategories = ["vintage-collectibles"]`, price range $10–$50, no excluded categories.
2. Wait for the session to reach `FRESHNESS_SCORED` (or `RECOMMENDATIONS_READY`).
3. Observe the outcome badge.

**Expected:** `outcome = NO_MATCH`. The `recommendations` list is empty. The `explanation` field contains a sentence that names the constraint that caused the failure (e.g., "No products in the catalog match the category 'vintage-collectibles'"). The UI renders an explanatory card rather than an empty list — the App UI's right pane shows the explanation text prominently. The freshness score chip is present; score may be 1 given no in-stock recommendations exist.

---

## J4 — Low freshness score is visually flagged

**Precondition:** Service is running. The seeded catalog is modified so that the `home-goods` category contains only out-of-stock, off-season products.

**Steps:**
1. Submit the "Home-goods renovator" seeded profile against the `home-goods` catalog snapshot.
2. Wait for the session to reach `FRESHNESS_SCORED`.
3. Observe the freshness score chip on the session card.

**Expected:** `freshness.score` is 1 or 2. The session card's border is highlighted red (matching the score-≤2 visual rule). The `freshness.rationale` explains the low score in terms of out-of-stock or off-season products ranked in the recommendation set.

---

## J5 — Apparel profile with brand preference yields partial match

**Precondition:** Service is running. The seeded apparel catalog contains EcoThread products but all are priced above $150. The shopper's price ceiling is $100.

**Steps:**
1. Submit a custom apparel profile with `preferredBrands = ["EcoThread"]`, price range $30–$100.
2. Wait for the session to reach `FRESHNESS_SCORED`.
3. Observe the outcome badge and recommendation list.

**Expected:** `outcome = PARTIAL_MATCH`. The recommendation list contains at least one product from the apparel category within the price range (from a non-EcoThread brand), since the preferred brand has no products in range. The `explanation` paragraph notes that no EcoThread products were found within the stated price range but alternative matches were found.
