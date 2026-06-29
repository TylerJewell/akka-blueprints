# User journeys — hr-assistant

## J1 — Submit a PTO query and get a cited answer

**Preconditions:** Service running on declared port (`http://localhost:9740/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9740/` → App UI tab.
2. From the **Seeded query** dropdown, pick `PTO inquiry`.
3. Enter `EM-001` in the Employee ID field.
4. Click **Submit query**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted query text (unchanged for this seed — no PII present). The employee profile chip shows department, title, hire date, and work location for `EM-001`. Both chip groups (PII and special-category) show empty or a "No sensitive fields detected" indicator.
- Within 30 s the card reaches `ANSWER_RECORDED`. The right pane shows: an applicability badge (`APPLICABLE`), the answer paragraph, and at least two citation rows — each with a `policyId` matching an entry in `policies.jsonl`, a `sectionReference`, and a `quotedPassage`.
- No status transitions to `FAILED` at any point.

## J2 — Guardrail blocks a malformed answer

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-hr-query.json` includes deliberately malformed entries.

**Steps:**
1. Submit any seeded query three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser network panel (`/api/queries/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed answer (per mock seed logic).
- The `before-agent-response` guardrail rejects it. The malformed answer NEVER lands in `QueryEntity` — there is no `AnswerRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed answer. The card transitions to `ANSWER_RECORDED` with an answer that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Special-category field never reaches the LLM

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. In the query textarea, type: `I have a chronic back condition and need to know if I qualify for remote-work accommodation under the disability policy.`
2. Leave Employee ID blank.
3. Click **Submit query**.
4. Wait for `ANSWER_RECORDED`.
5. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).
6. Fetch `GET /api/queries/{id}` and read `request.rawQueryText`.

**Expected:**
- The logged LLM call body contains `[REDACTED-SPECIAL-CATEGORY:health-condition]` in place of "chronic back condition". The phrase does not appear verbatim in the log.
- `request.rawQueryText` in the JSON still contains the original phrase — the audit log preserves it.
- `sanitized.specialCategoriesFound` lists `health-condition` (at minimum).
- The agent still returns a useful answer about remote-work accommodation policy, because the policy question itself ("qualify for remote-work accommodation") is not redacted.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. In the query textarea, type: `My email is jane.doe@acme.com and my personnel number is EID-887342. Can I carry over unused PTO?`
2. Leave Employee ID blank.
3. Click **Submit query**.
4. Wait for `ANSWER_RECORDED`.
5. Inspect the service log for the LLM call body.
6. Fetch `GET /api/queries/{id}` and read `request.rawQueryText`.

**Expected:**
- The logged LLM call body contains `[REDACTED-EMAIL]` and `[REDACTED-PID]` in place of the raw strings. Neither `jane.doe@acme.com` nor `EID-887342` appears.
- `request.rawQueryText` in the JSON retains the raw strings.
- `sanitized.piiCategoriesFound` lists `email` and `employee-id` (at minimum).
- The agent returns a normal PTO carry-over answer — the policy question is intact after sanitization.

## J5 — Answer with no matching policy cites NOT_APPLICABLE

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the query textarea, type a question that clearly falls outside the seeded policies: `What is the company car reimbursement rate for executives?`
2. Leave Employee ID blank.
3. Click **Submit query**.
4. Wait for `ANSWER_RECORDED`.

**Expected:**
- The applicability badge shows `NOT_APPLICABLE`.
- `answer.citations` is an empty list or contains no entries referencing a real `policyId` (the mock's NOT_APPLICABLE entries contain no citations by design).
- `answerText` directs the employee to contact HR directly.
- No `FAILED` state; the guardrail accepts an empty citations list paired with `NOT_APPLICABLE`.
