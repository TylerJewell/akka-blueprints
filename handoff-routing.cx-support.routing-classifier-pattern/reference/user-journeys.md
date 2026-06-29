# User journeys — routing-classifier-pattern

## J1 — General enquiry: end-to-end handoff to GeneralAgent

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `MessageSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated message seeded as a general enquiry to drop.

**Expected:**
- The message appears with status `RECEIVED` and within ~1 s transitions to `CLASSIFIED`. The route chip shows `GENERAL` and the confidence is `high` or `medium`.
- The centre column shows the route verdict block with a green "approved" indicator.
- Within ~10 s the right column populates with a `GeneralAgent` draft reply: a `Re: …` subject, 2–4 paragraphs, an action chip (`INFO_PROVIDED` or `FOLLOW_UP_SCHEDULED`), and a `general` specialist tag.
- The reply guardrail block shows a green "allowed" check.
- The status pill transitions to `PUBLISHED`. `finishedAt` is set.

## J2 — Refund request: end-to-end handoff to RefundAgent

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a refund-flavoured message (the seeded JSONL covers all categories in rotation).

**Expected:**
- Classifier emits `route = REFUND`. Route guardrail approves. Status `ROUTED_REFUND`.
- `RefundAgent` returns a `Reply` with `action = REFUND_INITIATED` or `INFO_PROVIDED`.
- Reply guardrail allows. Status `PUBLISHED`.
- Neither `GeneralAgent` nor `TechnicalAgent` is ever invoked for this message.

## J3 — Technical question: end-to-end handoff to TechnicalAgent

**Preconditions:** Same as J1.

**Steps:**
1. Wait for the simulator to drop a technical message.

**Expected:**
- Classifier emits `route = TECHNICAL`. Route guardrail approves. Status `ROUTED_TECHNICAL`.
- `TechnicalAgent` returns a `Reply` with `action = INFO_PROVIDED` or `ARTICLE_LINKED`.
- If `ARTICLE_LINKED`, the reply body contains a `[Article: HC…]` citation.
- Reply guardrail allows. Status `PUBLISHED`.

## J4 — Ambiguous message: short-circuits to ABANDONED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous message to drop.

**Expected:**
- Classifier emits `route = UNROUTABLE` with `confidence = low`.
- The workflow skips the route guardrail (UNROUTABLE goes directly to `abandonStep`).
- Status `ABANDONED`. The right column shows a muted "Abandoned — unroutable" block.
- No specialist is invoked; no `RouteVerdict` appears in the centre column.
- `abandonReason` is populated with the classifier's reason.

## J5 — Route guardrail blocks a low-confidence decision

**Preconditions:** The seeded JSONL includes a message that produces a low-confidence route decision in the mock responses.

**Steps:**
1. Wait for the relevant message to drop and be classified.

**Expected:**
- Classifier emits a route decision with `confidence = low` (or a route name not in the registry).
- The route guardrail returns `approved = false` with a `rejectionReason` of `"route-confidence-below-threshold"` or `"route-not-in-registry"`.
- Status `ROUTE_BLOCKED`. The centre column shows the route verdict block with a red "rejected" indicator and the rejection reason.
- The right column shows a red "Route blocked" notice. No specialist is ever invoked. The Unblock button is not shown (route-blocked messages have no operator override path).

## J6 — Reply guardrail blocks an out-of-policy draft

**Preconditions:** The seeded JSONL includes a message engineered to coax a specific-date refund promise from `RefundAgent` (and the mock-responses file includes a matching guardrail-tripping entry).

**Steps:**
1. Wait for the trip message to drop and be routed to `RefundAgent`.
2. Wait for `RefundAgent` to produce a draft.

**Expected:**
- Status reaches `REPLY_DRAFTED`.
- The reply guardrail block shows a red badge with the violation `invented-refund-date`.
- Status transitions to `REPLY_BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published), the violations list, and an Unblock button.
- Clicking Unblock, entering a note, and confirming sends `POST /api/messages/{id}/unblock`. The status transitions to `PUBLISHED` with the original draft as the final reply and the operator's note captured.
