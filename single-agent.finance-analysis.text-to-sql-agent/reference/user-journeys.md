# User journeys — text-to-sql-agent

Acceptance journeys that define when the generated system is considered working. Each journey maps to one of the acceptance criteria in `SPEC.md §10`.

---

## J1 — Happy-path aggregate query

**Preconditions:**
- The service is running (`/akka:build` completed, port 9505 is listening).
- The receipts.db is seeded with the 50 sample rows.
- A model-provider key is configured (or mock LLM is active).

**Steps:**
1. Open the App UI tab in a browser at `http://localhost:9505`.
2. Select "What is the total amount spent per merchant?" from the seeded question dropdown or type it into the Question field.
3. Enter `analyst-001` in the Submitted by field.
4. Click **Ask**.
5. Observe the new query card appear in the live list with status `SUBMITTED`.
6. Within 5 s (mock) or 30 s (live LLM), observe the card transition through `GENERATING` → `EXECUTING` → `EXECUTED` → `SANITIZED`.
7. Select the card to view its detail in the right pane.

**Expected:**
- The Generated SQL block shows a valid `SELECT merchant, SUM(...) ... GROUP BY merchant ...` statement.
- The Guardrail badge reads "passed".
- The result table shows one row per merchant with `total_amount` and `receipt_count` columns.
- No `[REDACTED-NAME]` or `[REDACTED-EMAIL]` tokens appear (this query does not select customer columns).
- The prose summary is a single sentence describing the top merchant.
- `GET /api/queries/{queryId}` returns a JSON body with `status: "SANITIZED"` and non-null `sanitized` and `rawResult` fields.

---

## J2 — SQL guardrail fires and agent retries

**Preconditions:**
- Mock LLM is active (option a during `/akka:specify`).
- The mock response file `generate-and-run-sql-query.json` contains a malformed entry with `DELETE FROM receipts` as the first entry for every 3rd query (modulo seed).
- This is the 3rd query submitted in the session (or `queryId` hash modulo 3 == 0).

**Steps:**
1. Submit any question (for example "List the 5 most expensive receipts.").
2. Watch the status pill.

**Expected:**
- The card passes through `GENERATING` and a Guardrail badge briefly reads "retried 1×" (yellow) during the retry.
- The card never reaches `FAILED`; the agent's second iteration produces a safe `SELECT` statement.
- The final Guardrail badge reads "retried 1× → passed" (yellow-to-green).
- The result table populates with receipt rows after the corrected SQL executes.
- `GET /api/queries/{queryId}` shows `status: "SANITIZED"` — the destructive SQL is never recorded as executed.

---

## J3 — PII is redacted from result rows

**Preconditions:**
- The service is running.
- A model-provider key is configured (or mock LLM returns a response whose result rows include `customer_name` and `customer_email` columns).

**Steps:**
1. Submit the question "Which customer had the highest single receipt amount in Q2?".
2. Wait for the card to reach `SANITIZED`.
3. Inspect the result table in the UI.
4. Call `GET /api/queries/{queryId}` directly (for example via curl or the browser address bar).

**Expected:**
- The UI result table shows `[REDACTED-NAME]` in the `customer_name` column and `[REDACTED-EMAIL]` in the `customer_email` column.
- The PII category chips above the table show `customer-name` and `email`.
- `GET /api/queries/{queryId}` JSON response shows `rawResult.rows[0].customer_name` as the original un-redacted name (for example `"Jane Doe"`) and `sanitized.rows[0].customer_name` as `"[REDACTED-NAME]"`.
- The SSE stream event for the `SANITIZED` transition carries only the sanitized rows.

---

## J4 — Question outside schema returns explanation, not a hallucinated answer

**Preconditions:**
- The service is running.
- A live LLM is configured (or the mock response file includes an "unanswerable" entry).

**Steps:**
1. Submit the question "What is the weather forecast for New York today?".
2. Wait for the card to reach `SANITIZED`.

**Expected:**
- The card reaches `SANITIZED` (not `FAILED`) — the agent does not crash when asked an unanswerable question.
- The Generated SQL block shows `SELECT 'The question cannot be answered from the receipts data.' AS message` or similar.
- The result table shows one row with a message column.
- The prose summary explains that the receipts schema does not contain weather data.
- No hallucinated merchant names or amounts appear in the result.

---

## J5 — SSE stream delivers live state transitions

**Preconditions:**
- The service is running.
- A second browser tab or `curl -N http://localhost:9505/api/queries/sse` is open before a query is submitted.

**Steps:**
1. Start an SSE listener: `curl -N http://localhost:9505/api/queries/sse`.
2. In the App UI, submit a new question.

**Expected:**
- The SSE listener receives one `query-update` event for each state transition: `SUBMITTED`, `GENERATING`, `EXECUTING`, `EXECUTED`, `SANITIZED`.
- Each event carries the full `QueryRow` JSON at the moment of that transition.
- The `SANITIZED` event's `sanitized.rows` field does not contain un-redacted customer identifiers.

---

## J6 — Concurrent queries run independently

**Preconditions:**
- The service is running.
- A live LLM (or mock) is configured.

**Steps:**
1. Submit query A: "What is the total amount spent per merchant?".
2. Immediately (within 1 s) submit query B: "How many receipts were submitted per month in 2026?".
3. Watch both cards in the live list.

**Expected:**
- Both cards progress through their lifecycle independently and in parallel.
- The status transitions for query A do not block query B (and vice versa).
- Both cards reach `SANITIZED` within 30 s of each other.
- The SQL shown in query A's detail panel is the merchant-aggregate query; query B's panel shows the monthly-count query.
- Neither query's result rows appear in the other's detail panel.
