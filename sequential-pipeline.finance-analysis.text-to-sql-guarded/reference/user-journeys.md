# User journeys — text-to-sql-guarded

## J1 — Submit a financial question and get a report

**Preconditions:** Service running on declared port (`http://localhost:9790/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded question `What were the top 10 vendors by spend last quarter?` has a matching parse entry and H2 seed data.

**Steps:**
1. Open `http://localhost:9790/` → App UI tab.
2. From the **Pick a seeded question** dropdown, pick `What were the top 10 vendors by spend last quarter?`.
3. Click **Run query**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PARSING` within 1 s more.
- Within ~20 s the card reaches `PARSED`. The right pane shows the SQL code block with a well-formed SELECT statement referencing the `transactions` table. No DDL or DML is present.
- Within ~20 s more the card reaches `QUERIED`. The right pane shows the column list and row count. Raw rows are not displayed.
- Within ~20 s more the card reaches `FORMATTED`. The right pane shows the narrative paragraph, a redaction summary chip, and the formatted results table with sanitized vendor names and spend totals.
- Total elapsed time: ≤ 60 s on the happy path.
- The guardrail rejection strip is empty (no rejections on this path).

## J2 — Safety halt blocks a destructive SQL statement

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `parse-question.json` includes one entry whose `sql` field begins with `DELETE FROM transactions`.

**Steps:**
1. Submit any seeded question four times in a row.
2. Watch the fourth submission's lifecycle in the browser dev tools network panel (`/api/queries/sse`).

**Expected:**
- On the fourth submission, the agent's PARSE task returns a `ParsedQuery` with `sql = "DELETE FROM transactions WHERE YEAR(transaction_date) = 2025"`.
- `SqlSafetyInspector.inspect(sql)` returns `safe = false, matchedPattern = "DELETE"` before `queryStep` starts.
- The workflow writes `SafetyHaltFired{sql, matchedPattern}` on the entity. The card transitions from `PARSING` directly to `HALTED`.
- The right pane shows a yellow warning banner: "Destructive SQL pattern detected: DELETE. Statement blocked before database contact."
- The service log contains no `QueryTools.executeSql` call for this queryId — the H2 database was never contacted.
- The card in the live list shows the orange flame icon.

## J3 — PII sanitizer redacts employee data from result rows

**Preconditions:** Mock LLM mode. The mock's `execute-query.json` includes one entry whose `rows` list contains an `employee_name` of "Jane Smith" and an `ssn` of "123-45-6789".

**Steps:**
1. Submit the seeded question `List all employees with salary above the department average`.
2. Wait for the card to reach `FORMATTED`.

**Expected:**
- The `rawResult` stored on the entity (accessible via the audit log) contains the raw row `{ employee_name: "Jane Smith", ssn: "123-45-6789", salary: "95000.00" }`.
- The `QueryReport` written to the entity carries the sanitized row `{ employee_name: "[NAME REDACTED]", ssn: "***-**-****", salary: "95000.00" }`.
- The right pane shows the redaction summary chip: "2 fields redacted" in purple.
- The `GET /api/queries/{id}` response contains only the sanitized report; the raw `RawResult` is not present in the response body.
- The service log shows a `PiiSanitizer.sanitize` call with `redactionCount = 2` before the FORMAT task started.

## J4 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM. The mock's `parse-question.json` includes one entry whose `tool_calls` array starts with `executeSql` (a QUERY-phase tool called during the PARSE phase).

**Steps:**
1. Submit any seeded question twice in a row.
2. Watch the second submission's lifecycle.

**Expected:**
- On the second submission's `parseStep`, the agent's first iteration calls `executeSql`. `SqlGuardrail` rejects it; a `GuardrailRejected{phase: "PARSE", tool: "executeSql", reason: "phase-violation: executeSql requires status in {PARSED, QUERYING} with parsedQuery present, saw PARSING"}` event lands on the entity.
- The rejected call NEVER reaches `QueryTools.executeSql` — no H2 query is executed on this iteration.
- The agent's second iteration calls `resolveSchema` (correct PARSE-phase tool). The card eventually reaches `FORMATTED` as in J1.
- The rejection-log strip on the card shows the one rejected call with its full structured reason.
- The small red dot is visible on the card in the live list.

## J5 — Schema-miss returns an empty report honestly

**Preconditions:** Service running. Any model provider. No H2 table matches the user's question.

**Steps:**
1. In the App UI, type a question that references a non-existent table (e.g., `Show me all blockchain asset positions`).
2. Click **Run query**.

**Expected:**
- `ParseTools.resolveSchema` returns an empty list (no matching tables in the in-process schema registry).
- The agent's PARSE task returns `ParsedQuery` with `sql = "-- no relevant tables found"` and `tablesReferenced = []`.
- `SqlSafetyInspector` passes the comment-only SQL (no destructive keyword). The workflow records `QuestionParsed` and advances to `queryStep`.
- `QueryTools.executeSql` returns a `RawResult` with `rows = []` and `rowCount = 0`.
- The `QueryReport.narrative` reads: "The query returned no rows."
- The card reaches `FORMATTED` with a grey "No PII detected" chip and an empty results table. Nothing crashes; the empty report is honestly empty.

## J6 — Concurrent queries complete independently

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit three different seeded questions in rapid succession without waiting for any to complete.

**Expected:**
- Three separate cards appear in the live list, each with its own `queryId` and independent status progression.
- Each card's right pane shows only that query's data — no cross-query mixing of SQL statements, results, or redaction summaries.
- All three reach `FORMATTED` (or `HALTED` if the mock triggers the destructive entry for one of them) within 3 × 60 s.
- The SSE stream delivers distinct events for each `queryId`; the UI reconciles by `queryId` attribute.
