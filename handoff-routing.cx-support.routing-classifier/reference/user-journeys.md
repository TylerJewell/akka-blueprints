# User journeys — routing-classifier

## J1 — Billing ticket: end-to-end handoff to BillingHandler

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `TicketSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the simulator to drop a billing-flavoured ticket.

**Expected:**
- The case appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject is not.
- Within ~10 s the classification block shows `category = BILLING`, `confidence = high`, and a one-sentence reason. The status pill changes to `CLASSIFIED` then `ROUTED_BILLING`.
- Within ~5 s of routing, the right column populates with the `BillingHandler` draft: a `Re: …` subject, a 3–5 paragraph body, and an action chip (typically `REFUND_INITIATED` or `INFO_PROVIDED`), with `handlerTag = "billing"`.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.
- Neither `TechnicalHandler` nor `AccountHandler` is invoked for this ticket.

## J2 — Technical ticket: end-to-end handoff to TechnicalHandler

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a technical-flavoured ticket.

**Expected:**
- Classification emits `category = TECHNICAL`. Status `ROUTED_TECHNICAL`.
- `TechnicalHandler` returns a `HandlerReply` with a concrete answer (`INFO_PROVIDED` or `ARTICLE_LINKED`), `handlerTag = "technical"`.
- Guardrail allows. Status `RESOLVED`.
- `BillingHandler` and `AccountHandler` are never invoked for this case.

## J3 — Account-management ticket: end-to-end handoff to AccountHandler

**Preconditions:** Same as J1.

**Steps:**
1. Wait for the simulator to drop an account-management ticket (credential reset, plan change, or cancellation request).

**Expected:**
- Classification emits `category = ACCOUNT`. Status `ROUTED_ACCOUNT`.
- `AccountHandler` returns a `HandlerReply` with `action = ACCOUNT_UPDATED` or `INFO_PROVIDED`, `handlerTag = "account"`.
- Guardrail allows. Status `RESOLVED`.
- `BillingHandler` and `TechnicalHandler` are never invoked for this case.

## J4 — Ambiguous ticket: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous ticket to drop.

**Expected:**
- Classifier emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `CaseEscalated`. Status `ESCALATED`.
- No handler is invoked; the right column shows a muted "Escalated — no handler invoked" block.
- `escalationReason` is populated with the classifier's reason.

## J5 — Guardrail blocks an out-of-policy draft

**Preconditions:** The seeded JSONL includes one ticket engineered to coax a same-day resolution promise from a handler (and the mock-responses file includes a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip ticket to drop and be classified (e.g., as `BILLING`).
2. Wait for the handler to produce a draft.

**Expected:**
- Status reaches `REPLY_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-resolution-timeline`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.

## J6 — Operator unblocks a previously-blocked case

**Preconditions:** A case in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked case.
2. Enter a note ("operator override — verified with team lead").
3. Confirm.

**Expected:**
- A `POST /api/cases/{id}/unblock` is sent.
- The case transitions to `RESOLVED`. The previously-blocked draft is now the published `HandlerReply`.
- The guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the operator's override action.
