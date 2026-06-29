# User journeys — akka-handoff-routing

## J1 — Billing conversation: end-to-end handoff to BillingSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `ConversationSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated turn seeded as billing-flavoured to drop.

**Expected:**
- The conversation appears with status `RECEIVED` and within ~1 s transitions to `FILTERED`. The filtered message excerpt is visible; the raw message is not.
- Within ~5 s the guardrail block in the centre column shows a green "passed" indicator. Status transitions to `GUARDRAIL_PASSED`.
- Within ~10 s the routing block shows `category = BILLING`, `confidence = high`, and a one-sentence reason. Status transitions to `ROUTED_BILLING`.
- Within ~5 s of routing, the right column populates with the `BillingSpecialist` reply: a 2–4 paragraph body, an action chip (typically `REFUND_INITIATED` or `INFO_PROVIDED`), and a `billing` specialist tag.
- Status pill transitions to `RESOLVED`. `finishedAt` is set.
- Within ~10 s of the routing decision, the handoff score chip shows a number 1–5 with a rationale.

## J2 — Technical conversation: end-to-end handoff to TechnicalSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a technical-flavoured turn (the seeded JSONL covers all categories in rotation).

**Expected:**
- Guardrail passes. Routing emits `category = TECHNICAL`. Status `ROUTED_TECHNICAL`.
- `TechnicalSpecialist` returns a `Reply` with a concrete answer (likely `INFO_PROVIDED` or `ARTICLE_LINKED`).
- Status `RESOLVED`.
- `BillingSpecialist` is never invoked for this conversation — no reply from it appears anywhere.

## J3 — Ambiguous conversation: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous turn to drop.

**Expected:**
- Guardrail passes (it checks safety, not ambiguity).
- Routing emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates with `ConversationEscalated`. Status `ESCALATED`.
- Neither specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the triage reason.
- A `HandoffScored` event still fires; the score chip appears (typically high, since the classifier was right to refuse).

## J4 — Routing guardrail blocks a conversation with a prompt-injection signal

**Preconditions:** The seeded JSONL includes one turn containing a prompt-injection signal engineered to trip the routing guardrail.

**Steps:**
1. Wait for the trip turn to drop.

**Expected:**
- Status transitions from `RECEIVED` to `FILTERED` as usual.
- `RoutingGuardrail` returns `allowed=false` with violation `prompt-injection-signal`.
- The workflow emits `HandoffBlocked`. Status `BLOCKED`.
- The centre column's guardrail block shows a red badge and the violation name.
- The right column shows the guardrail block only — **no specialist reply**, because the specialist was never called.
- An Unblock button is visible.
- `HandoffScored` does NOT fire (no `RoutingDecided` event was emitted).

## J5 — Operator unblocks a previously-blocked conversation

**Preconditions:** A conversation in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked conversation.
2. Enter a note ("verified manually — no actual injection intent").
3. Confirm.

**Expected:**
- A `POST /api/conversations/{id}/unblock` is sent.
- The conversation transitions to `RESOLVED`. The operator note is preserved in the audit record.
- The guardrail block continues to show the original violation list; a small note shows the operator's override.

## J6 — Handoff score appears on every routed conversation

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new conversation through the routing step.

**Expected:**
- Within ~10 s of `RoutingDecided`, the per-conversation handoff score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not prevent the reply from being published.
- Blocked conversations (`BLOCKED` / guardrail-rejected before `RoutingDecided`) have no score chip because no routing decision was ever made.
