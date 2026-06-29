# User journeys — airline-triage-router

## J1 — Booking request: end-to-end handoff to BookingSpecialist

**Preconditions:** Service running on port 9231; valid model-provider API key (or mock LLM); `RequestFeeder` enabled.

**Steps:**
1. Open `http://localhost:9231/` → App UI tab.
2. Wait up to 30 s for the feeder to drop a booking-flavoured request.

**Expected:**
- The request card appears with status `RECEIVED`, transitions to `SANITIZED` within ~1 s. The raw subject is not shown; the sanitized subject is visible. PII categories found chips appear if any redaction occurred.
- Within ~10 s, the classification block shows `intent = BOOKING`, `confidence = high`, and a one-sentence reason. Status changes to `CLASSIFIED` then `ROUTED_BOOKING`.
- Within ~5 s of routing, the right column populates with the `BookingSpecialist` draft: a `Re: …` subject, a 3–5 paragraph body, an outcome chip (`BOOKING_CONFIRMED` or `FOLLOW_UP_SCHEDULED`), and a `booking` specialist tag.
- The response guardrail block shows a green "allowed" check.
- Status transitions to `RESOLVED`. `finishedAt` is set.
- Within ~10 s of the classification event, the routing score chip shows a number 1–5 with a rationale.

## J2 — Baggage claim: end-to-end handoff to BaggageSpecialist

**Preconditions:** Same as J1; the seeded JSONL includes baggage-flavoured requests.

**Steps:**
1. Open the App UI tab.
2. Wait for the feeder to drop a baggage-claim-flavoured request (the JSONL covers intents in rotation).

**Expected:**
- Classification emits `intent = BAGGAGE`. Status `ROUTED_BAGGAGE`.
- `BaggageSpecialist` returns a `AirlineResolution` with `outcome = BAGGAGE_CLAIM_FILED` and a body that does not name a specific compensation amount.
- No tool-call guardrail block appears (baggage claims do not involve destructive tool actions in this scenario).
- Response guardrail allows. Status `RESOLVED`.
- `BookingSpecialist`, `ChangeSpecialist`, and `StatusSpecialist` are never invoked for this request.

## J3 — Ambiguous request: short-circuits to UNRESOLVED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous short message.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Classification emits `intent = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `RequestUnresolved`. Status `UNRESOLVED`.
- No specialist is invoked; the right column shows a muted "Unresolved — no specialist invoked" block with the `unresolvedReason`.
- A `RoutingScored` event fires independently; the score chip appears (typically high, since the classifier was correct to refuse).

## J4 — Before-tool-call guardrail blocks a non-refundable cancellation

**Preconditions:** The seeded JSONL includes a change request on a non-refundable fare; the mock-responses file includes a matching tool-call denial entry.

**Steps:**
1. Wait for the non-refundable-fare change request to drop and be classified as `CHANGE`.
2. Wait for `ChangeSpecialist` to propose a cancellation tool call.

**Expected:**
- Status reaches `ROUTED_CHANGE`.
- The tool-call guardrail block in the right column shows a red badge with `denialReason = "non-refundable-fare"`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the denial reason and an Unblock button.
- The response guardrail block does not appear (the block happened before the specialist returned a draft).

## J5 — Response guardrail blocks a draft that echoes a PNR token

**Preconditions:** The seeded JSONL includes a booking request where the specialist's mock response contains a PNR-shaped token.

**Steps:**
1. Wait for the booking request to drop and reach `RESOLUTION_DRAFTED`.

**Expected:**
- Status reaches `RESOLUTION_DRAFTED`.
- The response guardrail block shows a red badge with violation `"echoes-pnr-token"`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published to passenger), the violation list, and an Unblock button.
- Clicking Unblock, entering a note, and confirming sends `POST /api/requests/{id}/unblock`. The request transitions to `RESOLVED`; the previously-blocked draft is now the published response. The violation list remains visible on the audit record.

## J6 — Routing score appears on every classified request

**Preconditions:** Service running at least 60 s with the feeder enabled.

**Steps:**
1. Watch any new request through the classification step.

**Expected:**
- Within ~10 s of `IntentClassified`, the per-request routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- A low score does not block the workflow or change the specialist's execution — it is metadata only.
