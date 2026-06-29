# User journeys — memory-bank

## J1 — Store a fact and confirm it lands

**Preconditions:** Service running on declared port (`http://localhost:9876/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9876/` → App UI tab.
2. Set the operation toggle to **REMEMBER** and the namespace to `personal-assistant`.
3. Click **Load seeded example** to fill the content textarea with "My preferred meeting time is 9 AM" (seed entry 1).
4. Click **Store**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted content. If the seeded example contains any PII placeholder, PII category chips appear; otherwise the label reads "no PII found".
- Within 15 s the card reaches `STORED`. The right pane shows: the agent's acknowledgement sentence, and 2–4 extracted tags as chips (e.g., `meeting-time`, `morning-preference`, `9am`).

## J2 — Recall a previously stored fact

**Preconditions:** Service running with at least one STORED memory entry in the `personal-assistant` namespace (J1 completed, or seeded entries loaded).

**Steps:**
1. Set the operation toggle to **RECALL** and the namespace to `personal-assistant`.
2. Type "What time do I prefer for meetings?" into the content textarea.
3. Click **Recall**.

**Expected:**
- A new card appears in the live list for the RECALL operation with status `SUBMITTED`.
- The card transitions through `SANITIZED` → `STORING` → `STORED` within 15 s.
- The right pane shows a ranked match list. The entry from J1 (or the matching seed entry) appears first with a relevance score above 0.7.
- Each match shows the stored sanitized content, tags, and a timestamp.
- If no entries match, the list is empty and a "no relevant entries found" label appears.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider — the sanitizer runs either way.

**Steps:**
1. Set the operation toggle to **REMEMBER** and the namespace to `custom`.
2. In the content textarea, type: "Call me at +1-800-555-0199 or email me at john.doe@example.com about account ACC-9876543."
3. Click **Store**.
4. Wait for `STORED`.
5. Inspect the service log for the LLM call body.
6. Fetch `GET /api/memories/{id}` and read `request.rawContent`.

**Expected:**
- The logged LLM call body contains `[REDACTED-PHONE]`, `[REDACTED-EMAIL]`, and `[REDACTED-ACCOUNT]`. The raw strings do not appear.
- `request.rawContent` in the API response still contains the original text — the audit log preserves it.
- `sanitized.piiCategoriesFound` lists `phone`, `email`, `account-id` (at minimum).

## J4 — Recall against an empty namespace

**Preconditions:** Service running. The `project-notes` namespace has no stored entries (fresh start, or namespace not used).

**Steps:**
1. Set the operation toggle to **RECALL** and the namespace to `project-notes`.
2. Type any recall query (e.g., "What is the sprint goal?").
3. Click **Recall**.

**Expected:**
- The RECALL operation completes with status `STORED`.
- The right pane shows an empty match list.
- The agent's result includes `matches: []` and a `queryContent` echoing the query text.
- No error is surfaced — the empty namespace is a valid state.

## J5 — Cross-namespace isolation

**Preconditions:** Service running with at least one STORED entry in `personal-assistant` and at least one in `project-notes`.

**Steps:**
1. Store a fact in `personal-assistant` (e.g., "Preferred meeting time: 9 AM").
2. Store a different fact in `project-notes` (e.g., "Sprint goal: launch the beta API").
3. Issue a RECALL in `personal-assistant` for "meeting time".

**Expected:**
- The recall result contains only entries from the `personal-assistant` namespace.
- The `project-notes` entry ("Sprint goal: launch the beta API") does not appear in the matches, even if the query text would loosely match it.
- This confirms that namespace is respected as a filter in the workflow's context injection step.
