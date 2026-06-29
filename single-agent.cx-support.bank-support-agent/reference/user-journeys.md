# User journeys — bank-support-agent

## J1 — Submit a balance query and receive a low-risk response

**Preconditions:** Service running on declared port (`http://localhost:9303/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9303/` → App UI tab.
2. From the **Customer account** dropdown, pick `cust-0017` (a seeded checking account with no anomalies).
3. Select category `BALANCE_QUERY` and click **Load seeded example** to fill the enquiry text.
4. Click **Submit enquiry**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `ACCOUNT_LOADED` within 1 s. The right pane shows the masked account number, available balance, and `cardActive = true`.
- Within 30 s the card reaches `RESPONSE_RECORDED`. The right pane shows: a risk badge in the 1–3 (green) range, `blockCard = false`, tone chip `REASSURING`, and a 2–5-sentence answer referencing the account balance.
- Within 1 s of `RESPONSE_RECORDED`, the card reaches `LOGGED`. The sanitized log section appears with the redacted answer and no PII category chips (a balance query answer contains no direct identifiers beyond the masked account number).

## J2 — Tool-call guardrail blocks a premature card-block attempt

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `handle-enquiry.json` includes an entry for a BALANCE_QUERY that sets `blockCard=true` with `riskScore=2` (a malformed intent the guardrail must catch).

**Steps:**
1. Submit a balance query for `cust-0017` (same as J1, category BALANCE_QUERY).
2. Watch the workflow's `respondStep` in the service log.

**Expected:**
- The agent's first iteration calls `AccountLookupTool` with `blockCard=true` in the tool parameters.
- `ToolCallGuardrail` intercepts the call. The current task context shows `riskScore=2` (below threshold 7). The guardrail blocks the call and returns a structured error naming the unmet pre-condition.
- The agent replans: it calls `AccountLookupTool` with `blockCard=false` (a standard lookup) and produces a response with `blockCard=false` and `riskScore ≤ 3`.
- The service log shows one `guardrail.block` line for the vetoed call.
- The card reaches `LOGGED` with `blockCard=false`. No card-block action was triggered.

## J3 — Response guardrail rejects an out-of-range risk score

**Preconditions:** Service running with the mock LLM (`model-provider = mock`). The mock includes a malformed `handle-enquiry.json` entry with `riskScore=15`.

**Steps:**
1. Submit any seeded enquiry. The mock selects the malformed entry on the first iteration of every 3rd enquiry (modulo seed) — if needed, submit 2 warm-up enquiries first.
2. Monitor `/api/enquiries/sse` in the browser network panel.

**Expected:**
- The agent's first iteration returns a `SupportResponse` with `riskScore=15`.
- `ResponseGuardrail` rejects it. The malformed response NEVER lands in `EnquiryEntity` — there is no `ResponseRecorded` event with `riskScore=15`.
- The agent loop retries (iteration 2) and produces a valid response with `riskScore` in `[1, 10]`. The card transitions normally to `LOGGED`.
- The service log shows one `guardrail.reject` line for the first iteration with the structured-error code `"riskScore-out-of-range"`.

## J4 — PII redacted in audit log (sanitizer check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so log entries are visible. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a lost-card report for `cust-0041` (a seeded account whose mock response answer includes the masked account number `****4821` and the customer name `Alex Thornton`).
2. Wait for the card to reach `LOGGED`.
3. In the right-pane detail, inspect the **Audit log** section.
4. Fetch `GET /api/enquiries/{id}` directly (e.g., via the browser) and read `response.answer`.

**Expected:**
- The **Audit log** section in the UI shows the redacted answer: account number replaced with `[REDACTED-ACCOUNT]`, name replaced with `[REDACTED-NAME]`. The PII category chips show `account-number` and `name`.
- `GET /api/enquiries/{id}` returns the full entity including `response.answer` with the raw text intact (`****4821`, `Alex Thornton` still present).
- The view row (`EnquiryRow`) returned by `GET /api/enquiries` contains only `redactedAnswer` — no raw answer field.

## J5 — Lost-card report produces blockCard=true recommendation

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a lost-card report for `cust-0041` (category `LOST_CARD`).
2. Wait for `LOGGED`.

**Expected:**
- The `SupportResponse` carries `blockCard=true`, `riskScore ≥ 7`, and `tone=URGENT`.
- The right-pane detail highlights the blockCard indicator in red.
- The `before-tool-call` guardrail allowed the `AccountLookupTool` call with `blockCard=true` because the risk score was at or above threshold — confirmed by the absence of any `guardrail.block` line in the log for this enquiry.

## J6 — Unknown customerId returns a well-formed fallback response

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the Customer account field, type a `customerId` not present in the seeded accounts fixture (e.g., `cust-9999`).
2. Select any category and click **Submit enquiry**.
3. Wait for `LOGGED`.

**Expected:**
- `AccountLookupTool.lookup("cust-9999")` returns an empty result.
- The agent returns the standard fallback `SupportResponse` — `riskScore=5`, `blockCard=false`, `tone=NEUTRAL`, and an answer directing the customer to verify their customer ID.
- The card reaches `LOGGED`; no `FAILED` state.
- The UI right pane shows the fallback answer and an empty account summary section.
