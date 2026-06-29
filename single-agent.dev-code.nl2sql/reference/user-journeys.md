# User journeys — nl2sql

## J1 — Submit a question and get query results

**Preconditions:** Service running on declared port (`http://localhost:9643/`); Docker Compose Postgres container running (`docker compose up -d`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9643/` → App UI tab.
2. From the **Schema context** dropdown, pick `orders`.
3. Click **Load seed question** to fill the Question textarea with "What is total revenue per product this month?".
4. Click **Run query**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SCHEMA_ATTACHED` within 1 s. The right-pane detail shows the schema fragment listing the `orders`, `order_items`, and `products` tables with their columns.
- Within 30 s the card reaches `RESULT_READY`. The right pane shows: the generated SQL in a monospace block, a row count badge, and a data table with `name` and `revenue` columns and at least one row.
- Within 1 s of `RESULT_READY`, the card reaches `SCORED` and shows a score chip (1–5) plus a one-line rationale.

## J2 — Safety halt fires on write statement

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `run-query.json` includes an entry whose `generatedSql` begins with `DELETE`.

**Steps:**
1. Submit any seed question four times in a row (J1 steps × 4).
2. Watch the fourth submission's lifecycle in the network panel of the browser dev tools (`/api/queries/sse`).

**Expected:**
- The fourth submission's first agent iteration produces a `DELETE` statement.
- `SqlSafetyHalt` fires immediately. The task is terminated; no call reaches the Postgres container — there is no `ResultRecorded` event in the entity log for this query.
- The card transitions directly to `HALTED`. The right pane shows the red halt banner with the reason text: "Unsafe SQL detected: keyword DELETE at position 0. Task terminated."
- The service log shows one `halt.triggered` line naming the keyword and query id.

## J3 — Zero-row result receives low score

**Preconditions:** Mock LLM mode. The mock's `run-query.json` includes a zero-rows entry (empty `rows` array).

**Steps:**
1. Submit any seed question. The mock selects the zero-rows entry (deterministic by seed; trigger on the third query modulo seed).
2. Wait for `SCORED`.

**Expected:**
- The result lands well-formed (the guardrail checks SQL safety, not business logic).
- The score chip shows **2** and the rationale reads "Query returned no rows; confirm schema context is correct or broaden the date range."
- The card border highlights yellow. The user knows to inspect before acting on the result.

## J4 — Guardrail rejects SELECT * without LIMIT; agent self-corrects

**Preconditions:** Mock LLM mode. The mock includes a `SELECT *` entry without LIMIT.

**Steps:**
1. Submit any question three times in a row.
2. Watch the third submission's lifecycle in the network panel.

**Expected:**
- The third submission's first agent iteration calls `QueryExecutionTool` with `SELECT * FROM orders`.
- `SqlSafetyGuardrail` rejects it with the structured error "Forbidden pattern: SELECT * without LIMIT".
- The agent loop consumes one iteration and retries. The second iteration produces `SELECT order_id, customer_id, total_amount FROM orders LIMIT 1000`.
- The guardrail accepts the corrected query. `QueryExecutionTool` runs. The card reaches `RESULT_READY` and then `SCORED` as normal.
- The UI never displays the rejected query — only the accepted SQL appears in the right-pane detail.

## J5 — Schema lookup confirms column names before SQL generation

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the question "What is the average fulfillment time by region?" with the `fulfillment` schema context.
2. Wait for `SCORED`.

**Expected:**
- The agent's tool-call log shows at least one `SchemaLookupTool` call (for `fulfillment` and/or `orders`) before the `QueryExecutionTool` call.
- The generated SQL references only columns that exist in the schema fragment — no hallucinated column names.
- The result table contains a `region` column and an average-hours column. Score is 4 or 5.

## J6 — Full schema context routes across multiple tables

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Select `full schema` from the Schema context dropdown.
2. Submit "Show me the top 5 customers by total order value."
3. Wait for `SCORED`.

**Expected:**
- The schema fragment in the right pane lists all tables from the full schema (orders, order_items, customers, products, fulfillment).
- The generated SQL joins `customers` and `orders` (and optionally `order_items`) to compute a total.
- The result table contains `customer_id` or `name` and a total-order-value column with exactly 5 rows (LIMIT 5 or LIMIT 1000 with 5 rows returned).
- Score is 4 or 5; rationale confirms non-empty result and expected columns present.
