# User journeys — triage-handoff-routing

## J1 — Account intent: end-to-end handoff to AccountSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `ConversationSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated turn seeded as account-intent to drop.

**Expected:**
- The turn appears with status `RECEIVED` and within ~1 s transitions to `NORMALISED`. The normalised text is visible; the raw text is not shown to the user.
- Within ~10 s the triage block shows `intent = ACCOUNT`, `confidence = high`, and a one-sentence reason. The status pill changes to `TRIAGED`.
- Within ~3 s the routing verdict block appears with a green check ("allowed"). Status transitions to `ROUTED_ACCOUNT`.
- Within ~5 s of routing, the right column populates with the `AccountSpecialist` draft: a reply text, an action chip (typically `INFORMATION_PROVIDED` or `ACCOUNT_UPDATED`), and an "account" specialist tag.
- The response verdict block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.

## J2 — Product intent: end-to-end handoff to ProductSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a product-intent turn.

**Expected:**
- Triage emits `intent = PRODUCT`. Status `ROUTED_PRODUCT`.
- `ProductSpecialist` returns a `Reply` with a concrete answer (typically `INFORMATION_PROVIDED` or `FOLLOW_UP_SCHEDULED`).
- Response guardrail allows. Status `RESOLVED`.
- The `AccountSpecialist` and `ReturnSpecialist` are never invoked for this turn.

## J3 — Returns intent: end-to-end handoff to ReturnSpecialist

**Preconditions:** The seeded JSONL includes a returns-intent turn.

**Steps:**
1. Wait for the returns-intent turn to drop.

**Expected:**
- Triage emits `intent = RETURNS`. Status `ROUTED_RETURNS`.
- `ReturnSpecialist` returns a `Reply` with `RETURN_INITIATED` or `INFORMATION_PROVIDED`.
- Response guardrail allows. Status `RESOLVED`.
- The other specialists are never invoked.

## J4 — Routing guardrail blocks a low-confidence routing decision

**Preconditions:** The seeded JSONL includes a turn engineered to produce a low-confidence triage output (mixed-intent short message).

**Steps:**
1. Wait for the low-confidence turn to drop and be triaged.

**Expected:**
- Status reaches `TRIAGED`.
- Within ~3 s the routing verdict block appears with a red badge and the `reason` naming why it was blocked (e.g., "confidence low and intent mismatch").
- Status transitions to `ROUTING_BLOCKED`. No specialist is ever invoked.
- The right column shows a muted "Routing blocked — no specialist invoked" block with the routing verdict reason and a Review button.
- Clicking Review, entering a note, and choosing `publish=true` transitions the turn to `RESOLVED` (operator override).
- Clicking Review and choosing `publish=false` transitions the turn to `ESCALATED`.

## J5 — Response guardrail blocks an out-of-policy draft

**Preconditions:** The seeded JSONL includes a turn engineered to coax an invented policy citation out of a specialist.

**Steps:**
1. Wait for the trip turn to drop and be triaged and routed to the appropriate specialist.
2. Wait for the specialist to return a draft.

**Expected:**
- Status reaches `REPLY_DRAFTED`.
- Within ~3 s the response verdict block in the UI shows a red badge with the violation (e.g., `invented-policy-citation`).
- Status transitions to `RESPONSE_BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and a Review button.
- The routing verdict for the same turn remains green ("allowed") — the routing guardrail passed; only the response guardrail triggered.

## J6 — Ambiguous turn escalates without specialist invocation

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous turn to drop.

**Expected:**
- Triage emits `intent = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `TurnEscalated`. Status `ESCALATED`.
- The routing guardrail step is not reached (the workflow's escalate branch fires before `routingCheckStep`).
- Neither specialist is invoked; the right column shows a muted "Escalated — no reply sent" block.
- `escalationReason` is populated with the triage reason.
