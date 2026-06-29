# User journeys — ambient-expense-agent

## J1 — Submit a restaurant receipt and get a fully approved report

**Preconditions:** Service running on declared port (`http://localhost:9483/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9483/` → App UI tab.
2. From the **Load seeded example** dropdown, pick `Restaurant receipt`.
3. Enter a submitter ID (e.g., `employee-001`).
4. Click **Submit receipt**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted receipt text; PII category chips show at minimum `person-name` and `payment-card-number`.
- Within 30 s the card reaches `REPORT_READY`. The right pane shows: a line-item table with one row per charge on the receipt (entrée, wine), each with `APPROVED` status, a valid `categoryId` from the taxonomy, and the merchant name.
- The total-amount row matches the sum of the extracted line items.
- The report status badge shows `FULLY_APPROVED`.
- Within 2 s the card reaches `SUBMITTED_TO_SYSTEM`.

## J2 — Guardrail blocks a policy-exceeding line item

**Preconditions:** Service running. Mock LLM selected, or the hotel folio seed with the minibar charge present.

**Steps:**
1. From the **Load seeded example** dropdown, pick `Hotel folio`.
2. Enter a submitter ID.
3. Click **Submit receipt**.

**Expected:**
- The submission reaches `REPORT_READY`.
- The line-item table shows the room-charge rows as `APPROVED` and the minibar charge as `BLOCKED`.
- The `blockReason` column on the minibar row reads `amount-exceeds-limit: meals-and-entertainment maxAmountUsd 30 for alcohol sub-category` (or similar text naming the failed check).
- The report status badge shows `PARTIALLY_BLOCKED`.
- The service log contains one `guardrail.reject` line for the blocked tool call, naming the check that failed.
- The minibar charge's raw amount is preserved in `report.lineItems[].amount`; it is not silently dropped.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Paste a custom receipt into the textarea containing the literal strings `4111-1111-1111-1111` (a test PAN), `Jane Smith` (employee name), and `42 Elm Street, Springfield` (address).
2. Enter a submitter ID and click **Submit receipt**.
3. Wait for `REPORT_READY`.
4. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
5. Fetch `GET /api/expenses/{id}` and read `request.rawReceiptText`.

**Expected:**
- The logged LLM call body contains `[REDACTED-CARD]`, `[REDACTED-NAME]`, and `[REDACTED-ADDRESS]`. The raw strings do not appear.
- `request.rawReceiptText` in the JSON response still contains `4111-1111-1111-1111`, `Jane Smith`, and `42 Elm Street, Springfield`.
- `sanitized.piiCategoriesFound` lists `payment-card-number`, `person-name`, `address` (at minimum).

## J4 — Ambiguous voice transcript produces a partial report with FLAGGED items

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Paste the following voice transcript into the receipt textarea:
   ```
   I had about fifty bucks at some coffee place this morning, and then maybe a muffin or something, I think it was like six dollars separately or was it together, I'm not sure. Then taxi to the airport, thirty-two dollars.
   ```
2. Enter a submitter ID and click **Submit receipt**.

**Expected:**
- The submission reaches `REPORT_READY` without failing. The agent does not refuse the task.
- The line-item table contains at least two rows: the taxi charge as `APPROVED` (amount $32, category `ground-transport`, merchant `Taxi` or similar) and the coffee/food charge(s) as `FLAGGED` with the ambiguity described in `description`.
- The report status badge shows `FULLY_APPROVED` if only the taxi is actionable and the flagged items were not blocked, or `PARTIALLY_BLOCKED` if any items could not be categorized at all.
- No row in the line-item table has an empty `merchantName` unless genuinely absent from the transcript and the guardrail would have blocked an empty merchant.

## J5 — Multi-item receipt completeness

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Load seeded example** dropdown, pick `Travel itinerary` (flight + taxi + coffee — 3 distinct charges).
2. Submit and wait for `REPORT_READY`.

**Expected:**
- The report's `lineItems` array contains exactly 3 rows, one per visible charge on the itinerary.
- Each row has a `categoryId` present in the taxonomy (`air-travel`, `ground-transport`, `meals-and-entertainment`).
- No charge is silently omitted. The `totalAmount` matches the arithmetic sum of the three extracted amounts.
