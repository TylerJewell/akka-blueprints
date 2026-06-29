# User journeys — employee-helpdesk-router

## J1 — HR question: end-to-end handoff to HRSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `QuestionSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated question seeded as HR-flavoured to drop.

**Expected:**
- The question appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject is not. Any employee id or email in the body appears as `[REDACTED-EMP-ID]` or `[REDACTED-EMAIL]`.
- Within ~10 s the routing block shows `topic = HR`, `confidence = high`, and a one-sentence reason. The status pill changes to `ROUTED` then `ROUTED_HR`.
- Within ~5 s of routing, the right column populates with the `HRSpecialist` draft: a `Re: …` subject, a 3–5 paragraph body, an action chip (typically `INFO_PROVIDED` or `FOLLOW_UP_SCHEDULED`), and an `hr` specialist tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `ANSWERED`. `finishedAt` is set.
- Within ~10 s of the routing decision, the route score chip shows a number 1–5 with a rationale.

## J2 — IT question: end-to-end handoff to ITSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop an IT-flavoured question (the seeded JSONL covers HR, IT, POLICY, and UNCLEAR in rotation).

**Expected:**
- Routing emits `topic = IT`. Status `ROUTED_IT`.
- `ITSpecialist` returns an `Answer` with a concrete resolution (likely `INFO_PROVIDED` or `TICKET_OPENED`).
- Guardrail allows. Status `ANSWERED`.
- `HRSpecialist` and `PolicySpecialist` are never invoked for this question — no draft from either appears in any surface.

## J3 — Policy question: end-to-end handoff to PolicySpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Wait for the simulator to drop a policy-flavoured question.

**Expected:**
- Routing emits `topic = POLICY`. Status `ROUTED_POLICY`.
- `PolicySpecialist` returns an `Answer` that describes the relevant policy area without citing a specific document id.
- Guardrail allows. Status `ANSWERED`.

## J4 — Ambiguous question: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately short or ambiguous question.

**Steps:**
1. Wait for the ambiguous question to drop.

**Expected:**
- Routing emits `topic = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `QuestionEscalated`. Status `ESCALATED`.
- No specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the routing reason.
- A `RouteScored` event still fires; the score chip appears (typically high, since the classifier was correct to refuse).

## J5 — Guardrail blocks a fabricated policy reference

**Preconditions:** The seeded JSONL includes one question engineered to coax a fabricated policy document id out of the HR specialist (and the mock-responses file includes a matching trip-the-guardrail entry with `HR-POL-9991`).

**Steps:**
1. Wait for the trip question to drop and be routed to `HR`.
2. Wait for `HRSpecialist` to produce a draft citing `HR-POL-9991`.

**Expected:**
- Status reaches `ANSWER_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-policy-reference`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.

## J6 — HR-team unblocks a previously-blocked question

**Preconditions:** A question in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked question.
2. Enter a note ("HR lead reviewed — policy citation removed manually").
3. Confirm.

**Expected:**
- A `POST /api/questions/{id}/unblock` is sent.
- The question transitions to `ANSWERED`. The previously-blocked draft is now the published `Answer`.
- The guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the reviewer's override text.
