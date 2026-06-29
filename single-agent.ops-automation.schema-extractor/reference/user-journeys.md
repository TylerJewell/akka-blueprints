# User journeys — schema-extractor

## J1 — Submit an invoice and get a complete record

**Preconditions:** Service running on declared port (`http://localhost:9630/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9630/` → App UI tab.
2. From the **Target schema** dropdown, pick `Invoice`.
3. Click **Load seeded example** to fill the document title and the document textarea.
4. Click **Submit for extraction**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted document; the PII category chips show `email`, `address` (at minimum — depending on the seed content).
- Within 30 s the card reaches `RECORD_EXTRACTED`. The right pane shows: a schema-coverage badge of 100%, and a field row for each of the 10 declared Invoice schema fields. Every row has a non-empty `value` and a `confidence` of `HIGH` or `MEDIUM` (the seed document is designed to contain all required fields).

## J2 — Guardrail blocks a malformed record

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `extract-record.json` includes deliberately malformed entries (one missing a required field; one with a `fieldName` not in the submitted schema).

**Steps:**
1. Submit any seeded document three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/jobs/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed record.
- The `after-llm-response` guardrail rejects it. The malformed record NEVER lands in `ExtractionJobEntity` — there is no `RecordExtracted` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a valid record. The card transitions to `RECORD_EXTRACTED` with a record that satisfies all guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a document containing the literal strings `billing@acme.example`, `4111-1111-1111-1111` (a test payment-card number), and `+44 7911 123456`.
2. Wait for `RECORD_EXTRACTED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/jobs/{id}` and read `request.rawDocument`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-EMAIL]`, `[REDACTED-PCN]`, `[REDACTED-PHONE]`. The raw strings do not appear.
- `request.rawDocument` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `payment-card-number`, `phone` (at minimum).

## J4 — Required fields absent from document produce ABSENT confidence

**Preconditions:** Service running with mock LLM or a real model. A user manually submits a document that is clearly missing some required schema fields (e.g., a one-paragraph note with no dates or amounts).

**Steps:**
1. From the **Target schema** dropdown, pick `Invoice`.
2. Paste a short document that contains a vendor name but no dates, no amounts, and no invoice number.
3. Click **Submit for extraction**.
4. Wait for `RECORD_EXTRACTED`.

**Expected:**
- The extracted record lands with fields for `vendorName` present (confidence `HIGH` or `MEDIUM`) and the missing fields present with `confidence = ABSENT` and `value = ""`.
- The schema-coverage badge shows a percentage below 100% (the number of required fields with non-ABSENT confidence divided by total required fields).
- The coverage bar in the right pane is amber or red depending on the percentage.
- The guardrail did NOT reject the record — `ABSENT` confidence on a required field is a valid outcome. What the guardrail would have rejected is the field being omitted entirely.

## J5 — PurchaseOrder schema extraction end-to-end

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Target schema** dropdown, pick `PurchaseOrder`.
2. Click **Load seeded example**.
3. Click **Submit for extraction**.
4. Wait for `RECORD_EXTRACTED`.

**Expected:**
- The extracted record contains exactly 8 field entries matching the 8 fields declared in the PurchaseOrder schema — no extras, no omissions.
- Each `fields[i].fieldName` matches one of the 8 declared field names, one-to-one.
- The `schemaId` on the record is `purchase-order-schema`, matching the submitted schema.
- The sanitized document preview shows `[REDACTED-PHONE]` and `[REDACTED-ADDRESS]` markers where the seed document had raw values.
