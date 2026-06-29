# User journeys — reddit-search

## J1 — Submit a topic and receive a full report

**Preconditions:** Service running on declared port (`http://localhost:9988/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. `maxPages` left at default (10).

**Steps:**
1. Open `http://localhost:9988/` → App UI tab.
2. Type `distributed systems tradeoffs` into the **Topic** input.
3. Leave **Subreddit scope** blank (search all Reddit).
4. Click **Start research**.

**Expected:**
- A new job card appears in the live list with status `QUEUED` within 1 s.
- Within ~1 s the card transitions to `BROWSING` and the page-visit counter begins incrementing.
- Within 120 s the card reaches `REPORT_READY`. The right pane shows: a ranked post list with at least 3 entries, each with a non-empty `summaryLine` and a valid `url`; at least 2 theme entries; a sentiment bar with non-zero counts.
- `pagesVisited` on the finished card is between 1 and 10.

## J2 — Navigation guardrail blocks an off-domain URL

**Preconditions:** Service running with mock LLM. The mock `BrowserResearchAgent` is configured to issue at least one `navigate("http://reddit.com/...")` call (non-HTTPS) during its session.

**Steps:**
1. Submit any seeded topic with `maxPages=5`.
2. Wait for the job to reach `REPORT_READY` or `BUDGET_EXHAUSTED`.
3. Inspect the service log.

**Expected:**
- The log contains at least one `guardrail.reject: navigation.blocked: scheme must be https` line corresponding to the HTTP URL attempt.
- The job completes successfully (or hits the budget limit) — the guardrail rejection does not cause a `FAILED` transition. The agent continued with a corrected HTTPS URL on its next iteration.
- The final `ResearchReport.posts` does not include any entry whose `url` starts with `http://`.

## J3 — Budget halt fires and partial report is preserved

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the topic `PostgreSQL vs CockroachDB at scale` with `maxPages=2`.
2. Wait for the job card to stop updating.

**Expected:**
- The job card transitions to `BUDGET_EXHAUSTED`, not `FAILED`.
- The card shows `pagesVisited = 2` (the exact ceiling).
- The right pane displays a partial `ResearchReport` with whatever posts the agent had collected by the time the budget fired. The report may have 0, 1, or 2 post entries depending on timing — it is not empty if any page was successfully read.
- The amber "Page budget exhausted" banner is visible on the right pane.
- The service log shows one `BudgetSignal.EXCEEDED` line at page 2.

## J4 — No-results topic returns a clear reason

**Preconditions:** Mock LLM mode. The mock `research-topic.json` includes a no-results entry (empty `posts` list, non-empty `noResultsReason`).

**Steps:**
1. In the **Topic** input, type a string that maps to the no-results mock entry (the mock key is `no-results-test-topic`).
2. Click **Start research** and wait for `REPORT_READY`.

**Expected:**
- The job card reaches `REPORT_READY` (not `FAILED`).
- The right pane's post list area shows the `noResultsReason` string instead of a table.
- `ResearchReport.posts` is an empty list. `themes` is empty. `sentiment` is `{0,0,0}`.
- The job is not flagged as an error — no-results is a valid outcome, not a failure.

## J5 — Subreddit-scoped search restricts navigation

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the topic `Rust async runtime comparison` with `subredditScope = ["rust"]` and `maxPages=5`.
2. Wait for `REPORT_READY` or `BUDGET_EXHAUSTED`.

**Expected:**
- Every `PostSummary.url` in the report belongs to `https://www.reddit.com/r/rust/...` — no posts from other subreddits appear.
- The service log does not show any `navigation.blocked` lines for inter-Reddit navigation (the agent stayed within the allowed scope).
- The report has at least 1 post entry with a non-empty `summaryLine`.
