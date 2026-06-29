# User journeys — localwiki-ingest-agent

## J1 — Submit a URL and get a filed page

**Preconditions:** Service running on declared port (`http://localhost:9621/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9621/` → App UI tab.
2. Select source type `URL` and click **Load seeded example** to fill the source address and namespace fields with the seeded technical article URL and `/docs`.
3. Click **Submit for ingest**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the sanitized content preview; PII category chips show at minimum the categories found in the seed (e.g., `email`).
- Within 30 s the card reaches `PAGE_FILED`. The right pane shows: a page title, a one-paragraph summary, a markdown body with at least two `##` headings, 2–4 category tag chips, and a target path under `/docs/`.
- The target path begins with `/docs/` (not any forbidden prefix).

## J2 — Guardrail blocks a path traversal and agent corrects

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `ingest-source.json` includes a deliberately malformed entry whose `targetPath` starts with `../../../etc/passwd` (triggers the path-prefix check in G1). The mock selects this entry on the first iteration of every 3rd ingest (modulo seed).

**Steps:**
1. Submit ingest sources until the third submission triggers the malformed mock entry (submit three times from the App UI if using the seed).
2. Watch the third submission's lifecycle in the browser dev tools network panel (`/api/ingests/sse`).

**Expected:**
- The third submission's first agent iteration proposes `write_wiki_page` with `targetPath = "../../../etc/passwd"`.
- The `before-tool-call` guardrail rejects it. The tool NEVER executes — no page is written to the wiki at that path. There is no `PageFiled` event with the forbidden path.
- The agent loop retries on iteration 2 with a corrected `targetPath` under `/docs/`. The card transitions to `PAGE_FILED` with a well-formed page.
- The service log shows one `guardrail.reject` line for the rejected tool call, naming the `path-prefix-forbidden` check.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call attachment is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a URL whose simulated content contains the literal strings `alice.jones@example.com`, `SSN 456-78-9012`, and `4111-1111-1111-1111` (a test payment-card number).
2. Wait for `PAGE_FILED`.
3. Inspect the service log for the agent task attachment (`debug:agent.task.attachment`).
4. Fetch `GET /api/ingests/{id}` and read `submission.sourceAddress` (which holds the submitted address for audit).

**Expected:**
- The logged agent task attachment contains the redacted forms only: `[REDACTED-EMAIL]`, `[REDACTED-SSN]`, `[REDACTED-PCN]`. The raw strings do not appear in the attachment.
- The filed `WikiPage.body` does not contain any of the raw strings.
- `sanitized.piiCategoriesFound` lists `email`, `ssn`, `payment-card-number` (at minimum).

## J4 — Fetch failure produces a FAILED ingest with a clear reason

**Preconditions:** Service running. Mock LLM selected, with the mock configured to simulate an empty-body fetch on one of the seed sources.

**Steps:**
1. Submit the seeded PDF example (which the in-process stub is configured to return empty for this test path).
2. Wait for the status to stabilise.

**Expected:**
- The card transitions to `FAILED` within 15 s (the `awaitSanitizedStep` timeout).
- The entity's `failureReason` is `"fetch-failed: empty response"` (or similar; must be non-empty).
- The right pane shows the failure reason in a red callout.
- No `PageFiled` event exists on the entity. The entity's `page` field is `null` on the wire.
- The card border is red; the status pill shows `FAILED`.

## J5 — Image source produces a filed page with correct namespace

**Preconditions:** Service running with any model provider.

**Steps:**
1. Submit the seeded IMAGE example with target namespace `/blog`.
2. Wait for `PAGE_FILED`.

**Expected:**
- The filed page's `targetPath` begins with `/blog/`.
- The `categoryTags` list contains 2–4 entries relevant to the image's described content (none are generic tags like `article`).
- The summary is 1–2 sentences, derived from the image OCR/alt-text simulation.
- The `mimeType` in `sanitized` is `image/png` (or the MIME type of the simulated image).
