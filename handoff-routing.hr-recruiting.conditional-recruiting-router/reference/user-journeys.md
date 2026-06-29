# User journeys — conditional-recruiting-router

## J1 — Info-request email: end-to-end handoff to InfoRequester

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `EmailSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated email seeded as an info-request to drop.

**Expected:**
- The application appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject is not.
- Within ~10 s the routing block shows `route = INFO_REQUEST`, `confidence = high`, and a one-sentence reason. The status pill changes to `CLASSIFIED` then `ROUTED_INFO`.
- Within ~5 s of routing, the right column populates with the `InfoRequester` reply: a `Re: …` subject, a 2–4 paragraph body, an action chip (typically `QUESTION_ANSWERED` or `ARTICLE_LINKED`), and an `info-requester` specialist tag.
- The status pill transitions to `COMPLETED`. `finishedAt` is set.
- Within ~10 s of the routing decision, the routing score chip shows a number 1–5 with a rationale.

## J2 — Interview-request email: end-to-end handoff to InterviewOrganizer

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop an interview-request-flavoured email (the seeded JSONL covers routes in rotation).

**Expected:**
- Routing emits `route = INTERVIEW_REQUEST`. Status `ROUTED_INTERVIEW`.
- `InterviewOrganizer` calls `availability_lookup`, then calls `schedule_slot` with a valid future slot.
- The tool guardrail block shows a green "allowed" check.
- `CalendarConfirmation` appears in the right column: interview format, interviewer ID, proposed slot, `SLOT_BOOKED` outcome.
- Status `COMPLETED`. `finishedAt` set.
- `InfoRequester` is never invoked for this application — no reply from it appears in any surface.

## J3 — Unroutable email: short-circuits to UNROUTABLE_CLOSED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the unroutable email to drop.

**Expected:**
- Routing emits `route = UNROUTABLE` with `confidence = low`.
- The workflow terminates immediately with `ApplicationClosed`. Status `UNROUTABLE_CLOSED`.
- Neither specialist is invoked; the right column shows a muted "Unroutable — no specialist invoked" block.
- `closureReason` is populated with the routing reason.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier was right to refuse).

## J4 — Guardrail blocks a past-dated scheduling tool call

**Preconditions:** The seeded JSONL includes one email engineered to prompt a past-dated slot, and the mock-responses file includes a matching trip-the-guardrail `CalendarConfirmation` entry.

**Steps:**
1. Wait for the trip-the-guardrail email to drop and be classified `INTERVIEW_REQUEST`.
2. Wait for `InterviewOrganizer` to attempt a `schedule_slot` call.

**Expected:**
- Status reaches `ROUTED_INTERVIEW`.
- The tool guardrail block in the UI shows a red badge with the violation `past-dated-slot`.
- Status transitions to `TOOL_BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked tool-call details and an Unblock button.

## J5 — Recruiter unblocks a previously-blocked application

**Preconditions:** An application in `TOOL_BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked application.
2. Enter a note ("recruiter override — manually confirmed slot with interviewer").
3. Confirm.

**Expected:**
- A `POST /api/applications/{id}/unblock` is sent.
- The application transitions to `COMPLETED`. The previously-proposed slot is now the confirmed `CalendarConfirmation`.
- The tool guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the recruiter's override.

## J6 — Routing score appears on every classified application

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new application through the classification step.

**Expected:**
- Within ~10 s of `RoutingDecided`, the per-application routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the reply or calendar hold from being sent.
