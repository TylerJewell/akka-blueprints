# User journeys — auto-insurance-agent

## J1 — Claim request: end-to-end handoff to ClaimsSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `RequestSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as a claim-flavoured FNOL to drop.

**Expected:**
- The request appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw member identifiers are not.
- Within ~10 s the triage block shows `category = CLAIM`, `confidence = high`, and a one-sentence reason. The status pill changes to `TRIAGED` then `ROUTED_CLAIM`.
- Within ~5 s of routing, the right column populates with the `ClaimsSpecialist` draft: a `Re: …` subject, a 3–5 paragraph body, an action chip (typically `CLAIM_ACKNOWLEDGED` or `CLAIM_STATUS_PROVIDED`), and a `claims` specialist tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.
- Within ~10 s of the triage decision, the triage score chip shows a number 1–5 with a rationale.

## J2 — Roadside request: dispatch and confirmation reference

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a roadside-flavoured request (seeded JSONL covers all five categories in rotation).

**Expected:**
- Triage emits `category = ROADSIDE`. Status `ROUTED_ROADSIDE`.
- `RoadsideSpecialist` returns a `MemberResponse` with `action = ROADSIDE_DISPATCHED` and a `confirmationRef` in format `RSA-XXXXXXXX`.
- The right column displays the confirmation reference chip alongside the draft body.
- Guardrail allows. Status `RESOLVED`.
- No other specialist (`ClaimsSpecialist`, `PolicySpecialist`, `RewardsSpecialist`) is invoked for this request.

## J3 — Ambiguous request: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately off-topic one-liner.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Triage emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `RequestEscalated`. Status `ESCALATED`.
- No specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the triage reason.
- A `TriageScored` event still fires; the score chip appears (typically 4–5, since the classifier was correct to refuse ambiguous content).

## J4 — Guardrail blocks an invented settlement amount

**Preconditions:** The seeded JSONL includes one request engineered to coax an invented settlement figure out of `ClaimsSpecialist` (and the mock-responses file includes a matching guardrail-trip entry).

**Steps:**
1. Wait for the trip request to drop and be triaged as `CLAIM`.
2. Wait for `ClaimsSpecialist` to produce a draft.

**Expected:**
- Status reaches `RESPONSE_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-settlement-amount`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.
- The specific dollar amount does not appear in the "published" surface; it is only visible in the blocked draft section.

## J5 — Supervisor unblocks a previously-blocked request

**Preconditions:** A request in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked request.
2. Enter a supervisor note (e.g., "reviewed with claims lead — amount pre-authorised by adjuster").
3. Confirm.

**Expected:**
- A `POST /api/requests/{id}/unblock` is sent.
- The request transitions to `RESOLVED`. The previously-blocked draft is now the published `MemberResponse`.
- The guardrail block continues to show the original violation list (the audit record is preserved); a note shows the supervisor's override text.

## J6 — Triage score appears on every triaged request

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new request through the triage step.

**Expected:**
- Within ~10 s of `TriageDecided`, the per-request triage score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the response from being published.
