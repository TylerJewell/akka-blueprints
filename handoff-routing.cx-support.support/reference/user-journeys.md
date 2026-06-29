# User journeys — support

## J1 — Billing ticket: end-to-end handoff to BillingSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `RequestSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as billing-flavoured to drop.

**Expected:**
- The ticket appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject is not.
- Within ~10 s the triage block shows `category = BILLING`, `confidence = high`, and a one-sentence reason. The status pill changes to `TRIAGED` then `ROUTED_BILLING`.
- Within ~5 s of routing, the right column populates with the `BillingSpecialist` draft: a `Re: …` subject, a 3–5 paragraph body, an action chip (typically `REFUND_INITIATED` or `INFO_PROVIDED`), and a `billing` specialist tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.
- Within ~10 s of the triage decision, the triage score chip shows a number 1–5 with a rationale.

## J2 — Technical ticket: end-to-end handoff to TechnicalSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a technical-flavoured request (the seeded JSONL covers the three categories in rotation).

**Expected:**
- Triage emits `category = TECHNICAL`. Status `ROUTED_TECHNICAL`.
- `TechnicalSpecialist` returns a `Resolution` with a concrete answer (likely `INFO_PROVIDED` or `ARTICLE_LINKED`).
- Guardrail allows. Status `RESOLVED`.
- The `BillingSpecialist` is never invoked for this ticket — no draft from it appears in any surface.

## J3 — Ambiguous ticket: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Triage emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `TicketEscalated`. Status `ESCALATED`.
- Neither specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the triage reason.
- A `TriageScored` event still fires; the score chip appears (typically high, since the classifier was right to refuse).

## J4 — Guardrail blocks an out-of-policy draft

**Preconditions:** The seeded JSONL includes one request engineered to coax a 24-hour refund promise out of the specialist (and the mock-responses file includes a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip request to drop and be triaged as `BILLING`.
2. Wait for `BillingSpecialist` to produce a draft.

**Expected:**
- Status reaches `RESOLUTION_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-refund-timeline`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.

## J5 — Operator unblocks a previously-blocked ticket

**Preconditions:** A ticket in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked ticket.
2. Enter a note ("operator override — verified with billing lead").
3. Confirm.

**Expected:**
- A `POST /api/tickets/{id}/unblock` is sent.
- The ticket transitions to `RESOLVED`. The previously-blocked draft is now the published `Resolution`.
- The guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the operator's override.

## J6 — Triage score appears on every triaged ticket

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new ticket through the triage step.

**Expected:**
- Within ~10 s of `TriageDecided`, the per-ticket triage score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the resolution from being published.
