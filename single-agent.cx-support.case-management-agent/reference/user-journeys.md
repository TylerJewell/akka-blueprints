# User journeys — case-management-agent

Six numbered acceptance journeys. Each defines the preconditions, steps, and expected outcome that `reference/user-journeys.md` uses as the acceptance bar for the generated system.

---

## J1 — Happy path: new billing-dispute case

**Preconditions:** No existing case for customer `cust-demo-1`. Service is running. Mock or live LLM available.

**Steps:**
1. POST `/api/messages` with `{ customerId: "cust-demo-1", rawText: "I was charged twice this month for my subscription. My card ending 1234 was billed on June 1 and June 3.", channel: "WEB_CHAT" }`.
2. Receive `{ messageId, caseId }`.
3. Poll `GET /api/cases/{caseId}` or watch the SSE stream.

**Expected:**
- Within ~1 s, `status` transitions from `OPEN` → sanitized message visible, `piiCategoriesFound` includes `payment-card`.
- Within ~30 s, `status` is `OPEN` or `IN_PROGRESS`, `lastAction.actionType == "CREATE"`, `lastAction.category == "billing"`, `lastAction.priority` is `MEDIUM` or `HIGH`.
- `eval.score` is populated (1–5); card displays without errors.
- `lastMessage.rawText` contains the original card reference; `sanitizedMessage.redactedText` contains `[REDACTED-PAYMENT-CARD]`.

---

## J2 — Guardrail retry: missing priority field

**Preconditions:** Mock LLM configured. Service is running. Mock `handle-customer-message.json` contains at least one entry missing the `priority` field.

**Steps:**
1. POST `/api/messages` with a message whose seed triggers a malformed response on iteration 1 (every 3rd message by mock seed).
2. Watch the SSE stream.

**Expected:**
- The case transitions to `IN_PROGRESS` without the malformed action ever appearing in the entity log or the UI.
- `lastAction` reflects the second iteration's well-formed response.
- No `CaseFailed` event; the case reaches `OPEN` or `IN_PROGRESS` normally.
- The malformed action is never visible in `GET /api/cases/{id}`.

---

## J3 — Tool-call guardrail: tier-skip escalation blocked

**Preconditions:** An existing TIER_1 case `case-tier1` is in `IN_PROGRESS`. Mock LLM configured to produce a TIER_3 escalation on this case id.

**Steps:**
1. POST `/api/messages` with `{ customerId, existingCaseId: "case-tier1", rawText: "The outage is affecting all our users. We need engineering." }`.
2. Watch the SSE stream.

**Expected:**
- The `ToolCallGuardrail` rejects the TIER_3 escalation with a policy-violation reason.
- The agent retries with a TIER_2 escalation.
- `lastAction.tier == "TIER_2"` and `lastAction.actionType == "ESCALATE"` appear in the entity log.
- The case transitions to `ESCALATED`.
- No `CaseFailed` event is emitted.

---

## J4 — PII never reaches the LLM

**Preconditions:** Service is running. LLM call logging is enabled (dev-mode default).

**Steps:**
1. POST `/api/messages` with `{ customerId: "cust-pii-test", rawText: "My name is Jane Doe. Email me at jane.doe@example.com. My account is ACT-9988.", channel: "EMAIL" }`.
2. After the case reaches `IN_PROGRESS`, inspect the LLM call log.
3. Fetch `GET /api/cases/{caseId}`.

**Expected:**
- LLM call log contains only `[REDACTED-EMAIL]`, `[REDACTED-PERSON-NAME]`, and `[REDACTED-ACCOUNT-ID]` — not the original strings.
- `GET /api/cases/{caseId}` response has `lastMessage.rawText` with the raw values intact.
- `sanitizedMessage.piiCategoriesFound` includes `email`, `person-name`, and `account-id`.

---

## J5 — Case lifecycle: create → update → close

**Preconditions:** Service is running. Live or mock LLM available.

**Steps:**
1. POST `/api/messages` → new case `case-flow` created with `actionType: CREATE`.
2. POST `/api/messages` with `{ existingCaseId: "case-flow", rawText: "I have more details about the billing issue." }` → case updated.
3. POST `/api/messages` with `{ existingCaseId: "case-flow", rawText: "The issue is resolved, thank you." }` → case closed.

**Expected:**
- After step 1: `status == "OPEN"`, `lastAction.actionType == "CREATE"`.
- After step 2: `status == "IN_PROGRESS"`, `lastAction.actionType == "UPDATE"`.
- After step 3: `status == "RESOLVED"`, `lastAction.actionType == "CLOSE"`, `resolvedAt` is populated.
- All three `EvaluationScored` events appear in the entity log; all three eval scores are visible in `GET /api/cases/{caseId}` history.

---

## J6 — Eval flags low-quality action

**Preconditions:** Mock LLM configured. A mock entry with blank `agentReasoning` is available.

**Steps:**
1. POST `/api/messages` with a seed that produces a `CaseAction` with an empty `agentReasoning` field (bypassing ActionGuardrail, which only checks structural validity).
2. Wait for `EvaluationScored`.

**Expected:**
- `eval.score <= 2`.
- `eval.rationale` mentions the missing reasoning field.
- The UI flags the case card with a red border.
- The case itself reaches `OPEN` or `IN_PROGRESS` normally — the eval is non-blocking.
