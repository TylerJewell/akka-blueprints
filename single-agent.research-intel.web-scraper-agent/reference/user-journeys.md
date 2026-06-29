# User journeys — web-scraper-agent

## J1 — Submit an allowed URL and get a structured extraction

**Preconditions:** Service running on declared port (`http://localhost:9519/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9519/` → App UI tab.
2. From the **Seeded URL** dropdown, pick `news-article`.
3. Confirm the **Target URL** input pre-fills with `https://news.example.net/transit-plan-approved` and the **Extraction schema** dropdown shows `news-article`.
4. Click **Submit scrape**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `FETCHING` within 1 s. The right pane shows the target URL and the extraction schema.
- Within 30 s (live LLM) or 3 s (mock), the card reaches `EXTRACTED`. The right pane shows the agent's title, summary, and a data-points table with at least 3 rows, each with a non-empty `sourceQuote`.
- Within 1 s of `EXTRACTED`, the card transitions to `SANITIZED`. The PII category chips appear above the summary. Any redacted tokens in the data-points table are highlighted in muted red.

## J2 — Blocked domain is rejected without an outbound fetch

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the **Target URL** input, type `https://restricted.internal.example.com/data`.
2. Select any extraction schema.
3. Click **Submit scrape**.

**Expected:**
- The card appears with status `SUBMITTED`, then immediately transitions to `BLOCKED` (within 1–2 s, before any `FETCHING` state is visible for more than a moment).
- A red banner appears in the right pane with the blocked reason: `"domain-not-allowed: restricted.internal.example.com"`.
- The service log shows one `guardrail.reject` line with `reason=domain-not-allowed` and zero lines indicating an outbound HTTP call to that host.
- `GET /api/scrapes/{id}` returns `"status": "BLOCKED"`, `"blockedReason": "domain-not-allowed: restricted.internal.example.com"`, `"result": null`.

## J3 — PII in extracted content is redacted before storage

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider. The `news-article` fixture (`news.example.net/transit-plan-approved`) contains a reporter byline email and a contact phone number.

**Steps:**
1. Submit the `news-article` seeded URL (J1 steps).
2. Wait for `SANITIZED`.
3. Inspect the right pane's data-points table.
4. Fetch `GET /api/scrapes/{id}` and read `result.dataPoints`.

**Expected:**
- The right pane's data-points table shows `[REDACTED-EMAIL]` and `[REDACTED-PHONE]` tokens in the relevant cells. The raw email and phone number strings do not appear in the UI.
- `sanitized.piiCategoriesFound` contains `"email"` and `"phone"` (at minimum).
- `GET /api/scrapes/{id}` returns `result.dataPoints` with the original values intact — the audit copy is unredacted on the entity.

## J4 — Rate-limit blocks excess scrapes from the same origin

**Preconditions:** Service running. Mock LLM (fast turnaround). Rate-limit window is 60 s, budget is 5 per origin.

**Steps:**
1. Submit the `news-article` seeded URL (`https://news.example.net/...`) five times in rapid succession (within the same 60-second window).
2. Submit a sixth scrape to the same origin.

**Expected:**
- Scrapes 1–5 each transition through `FETCHING → EXTRACTED → SANITIZED` normally.
- The sixth scrape transitions to `BLOCKED` with `blockedReason` containing `"rate-limit-exceeded: news.example.net"`.
- `GET /api/scrapes/{id}` for the sixth scrape returns `"status": "BLOCKED"` and `"result": null`.
- After the 60-second window resets, a new submission to `news.example.net` proceeds normally.

## J5 — Extraction from a product documentation page

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Seeded URL** dropdown, pick `product-doc`.
2. Confirm the extraction schema switches to `product-doc`.
3. Click **Submit scrape**.

**Expected:**
- The card reaches `SANITIZED` and the data-points table shows at minimum: `productName`, `version`, `description`. Each row has a non-empty `sourceQuote`.
- No `BLOCKED` or `FAILED` state occurs — `products.example.co` is in the seeded allowlist.
- The `httpStatus` field in `GET /api/scrapes/{id}` → `result.httpStatus` is `200`.
