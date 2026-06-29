# User journeys — triage-router-ui

## J1 — Billing message: end-to-end handoff to BillingHandler

**Preconditions:** Service running on port 9573; valid model-provider API key set (or mock LLM); `MessageSimulator` enabled.

**Steps:**
1. Open `http://localhost:9573/` → App UI tab.
2. Wait up to 30 s for the first simulated billing-flavoured message to drop.

**Expected:**
- The message appears with status `RECEIVED`. Within ~1 s the routing block shows `category = BILLING`, `confidence = high`, and a one-sentence reason. The status pill changes to `ROUTED_BILLING`.
- Within ~5 s of routing, the right column populates with the `BillingHandler` draft: a `Re: …` subject, a 3–5 paragraph body, an action chip (typically `REFUND_INITIATED` or `INFO_PROVIDED`), and a `billing` handler tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.

## J2 — Product message: end-to-end handoff to ProductHandler

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a product-flavoured message.

**Expected:**
- Router emits `category = PRODUCT`. Status `ROUTED_PRODUCT`.
- `ProductHandler` returns a `HandlerReply` with a concrete answer (likely `INFO_PROVIDED` or `ARTICLE_LINKED`).
- Guardrail allows. Status `RESOLVED`.
- `BillingHandler` is never invoked for this message — no draft from it appears in any surface.

## J3 — Ambiguous message: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous or very short message.

**Steps:**
1. Wait for the ambiguous message to drop.

**Expected:**
- Router emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `MessageEscalated`. Status `ESCALATED`.
- Neither handler is invoked; the right column shows a muted "Escalated — no handler invoked" block.
- `escalationReason` is populated with the routing reason.

## J4 — Guardrail blocks an out-of-policy draft

**Preconditions:** The seeded JSONL includes one message engineered to coax a next-business-day refund promise out of the billing handler (and the mock-responses file includes a matching guardrail-tripping entry).

**Steps:**
1. Wait for the trip message to drop and be routed as `BILLING`.
2. Wait for `BillingHandler` to produce a draft.

**Expected:**
- Status reaches `REPLY_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-refund-timeline`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.

## J5 — Operator unblocks a previously-blocked message

**Preconditions:** A message in `BLOCKED` state (see J4).

**Steps:**
1. Click the Unblock button on the blocked message in the right column.
2. Enter a note ("verified with billing lead — timeline acceptable").
3. Confirm.

**Expected:**
- A `POST /api/messages/{id}/unblock` is sent with `decidedBy` and `note`.
- The message transitions to `RESOLVED`. The previously-blocked draft is now the published `HandlerReply`.
- The guardrail block continues to show the original violation list (audit record preserved); a small note shows the operator override text.
- `finishedAt` is now set.
