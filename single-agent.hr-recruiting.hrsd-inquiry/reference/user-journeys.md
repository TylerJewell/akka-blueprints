# User journeys — hrsd-inquiry

## J1 — Submit a benefits inquiry and get a policy-cited answer

**Preconditions:** Service running on declared port (`http://localhost:9222/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9222/` → App UI tab.
2. From the **Topic** dropdown, pick `Benefits`.
3. Click **Load seeded example** to fill the employee ID and message textarea.
4. Leave the service-request checkbox unchecked.
5. Click **Ask HR**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SCREENED` within 1 s. The right-pane detail shows the screened message; if the seeded example contains a special-category phrase, the category chip appears (e.g., `health-condition`).
- Within 30 s the card reaches `ANSWERED`. The right pane shows: the answer paragraph, at least one cited policy chip (e.g., `BEN-101`), and no service-request section (since the checkbox was unchecked or the agent did not produce one for this topic).
- The card stays at `ANSWERED` and does not advance to `REQUEST_SUBMITTED`.

## J2 — Guardrail blocks an uncited answer

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-hr-inquiry.json` includes a deliberately malformed entry with an empty `citedPolicies` list.

**Steps:**
1. Submit any seeded inquiry three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/inquiries/sse`).

**Expected:**
- The third submission's first agent iteration produces a response with an empty `citedPolicies` list.
- The `before-agent-response` guardrail rejects it. The uncited response NEVER lands in `InquiryEntity` — there is no `InquiryAnswered` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a response with at least one policy citation. The card transitions to `ANSWERED` with a properly-cited answer.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code indicating which check failed (`citedPolicies-empty`).

## J3 — Special-category data never reaches the LLM

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider.

**Steps:**
1. Submit an inquiry with `rawMessage` containing `"I have Type 2 diabetes and need to understand my accommodation options."` and `topic = GENERAL_HR`.
2. Wait for `ANSWERED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).
4. Fetch `GET /api/inquiries/{id}` and read `request.rawMessage`.

**Expected:**
- The logged LLM call body contains only `[REDACTED-HEALTH]` where `"Type 2 diabetes"` appeared — e.g., `"I have [REDACTED-HEALTH] and need to understand my accommodation options."` The raw phrase does not appear.
- `request.rawMessage` in the JSON still contains the original phrase — the audit log preserves it.
- `screened.specialCategoriesFound` lists `health-condition`.

## J4 — Service request is submitted with a confirmation reference

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the Topic dropdown, pick `Leave & Absence`.
2. Click **Load seeded example** (a seeded leave inquiry that prompts the agent to produce a `LEAVE_REQUEST` service request).
3. Check the **Submit HR service request if applicable** checkbox.
4. Click **Ask HR**.
5. Wait for the card to reach its terminal state.

**Expected:**
- The card advances through `ANSWERED` → `REQUEST_SUBMITTED`.
- The right pane's service-request section shows `requestType: LEAVE_REQUEST`, the description, and the fields table.
- The confirmation reference chip appears in the format `HR-xxxxxxxx-xxxxx` (8 uppercase chars from the inquiryId, then a 5-digit number).
- `serviceRequestRef` in `GET /api/inquiries/{id}` matches the chip text.

## J5 — Inquiry with no applicable service request stays at ANSWERED

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit an inquiry with a pure informational question (e.g., "When does open enrollment end?") and `topic = BENEFITS`.
2. Check the **Submit HR service request if applicable** checkbox.
3. Click **Ask HR**.

**Expected:**
- The agent produces an `InquiryResponse` with a null `serviceRequest` (this is a question about dates, not an actionable transaction).
- The card reaches `ANSWERED` and stops — it does NOT advance to `REQUEST_SUBMITTED`, even though the checkbox was checked.
- The service-request section is absent from the right pane.

## J6 — Multi-policy inquiry cites every relevant policy

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit an inquiry asking about both payroll correction and the deadline to dispute a paycheck, `topic = PAYROLL`.
2. Wait for `ANSWERED`.

**Expected:**
- The `citedPolicies` list contains at least two entries, covering both the correction process policy and the dispute-deadline policy.
- The guardrail confirmed the response was well-formed (no rejection lines in the log).
- Each policy chip in the UI matches an id that exists in `src/main/resources/policy-catalog/payroll.txt`.
