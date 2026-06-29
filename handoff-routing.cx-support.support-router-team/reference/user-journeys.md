# User journeys — support-multi-agent

## J1 — Billing case: end-to-end handoff to BillingSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `TicketSimulator` enabled.

**Steps:**
1. Open `http://localhost:9427/` → App UI tab.
2. Wait up to 30 s for the first simulated case seeded as billing-flavoured to drop.

**Expected:**
- The case appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject is not.
- Within ~10 s the routing block shows `category = BILLING`, `confidence = high`, and a one-sentence reason. The status pill changes to `ROUTED_BILLING`.
- Within ~5 s of routing, the right column populates with the `BillingSpecialist` draft: a `Re: …` subject, a 3–5 paragraph body, an action chip (typically `REFUND_INITIATED` or `INFO_PROVIDED`), and a `billing` specialist tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.

## J2 — Technical case: end-to-end handoff to TechnicalSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a technical-flavoured case (the seeded JSONL covers all categories in rotation).

**Expected:**
- Routing emits `category = TECHNICAL`. Status `ROUTED_TECHNICAL`.
- `TechnicalSpecialist` returns a `Resolution` with a concrete answer (likely `INFO_PROVIDED` or `ARTICLE_LINKED`).
- Guardrail allows. Status `RESOLVED`.
- `BillingSpecialist` and `AccountSpecialist` are never invoked for this case — no draft from either appears in any surface.

## J3 — Account management case: end-to-end handoff to AccountSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Wait for the simulator to drop an account-flavoured case (e.g. a locked-out request).

**Expected:**
- Routing emits `category = ACCOUNT`. Status `ROUTED_ACCOUNT`.
- `AccountSpecialist` returns a `Resolution` with action `ACCESS_RESTORED` or `INFO_PROVIDED`.
- Guardrail allows. Status `RESOLVED`.
- `BillingSpecialist` and `TechnicalSpecialist` are not invoked.

## J4 — Ambiguous case: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Routing emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `CaseEscalated`. Status `ESCALATED`.
- No specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the routing reason.

## J5 — Guardrail blocks an out-of-policy draft

**Preconditions:** The seeded JSONL includes one request engineered to coax a 24-hour refund promise out of the billing specialist (and the mock-responses file includes a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip request to drop and be routed as `BILLING`.
2. Wait for `BillingSpecialist` to produce a draft.

**Expected:**
- Status reaches `RESOLUTION_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-refund-timeline`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.

## J6 — Operator unblocks a previously-blocked case

**Preconditions:** A case in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked case.
2. Enter a note ("operator override — confirmed with billing lead").
3. Confirm.

**Expected:**
- A `POST /api/cases/{id}/unblock` is sent.
- The case transitions to `RESOLVED`. The previously-blocked draft is now the published `Resolution`.
- The guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the operator's override.
