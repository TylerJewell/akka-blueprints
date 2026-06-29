# User journeys — cx-handoff-triage

## J1 — Sales conversation: end-to-end handoff to SalesSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `ConversationSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated conversation seeded as sales-flavoured to drop.

**Expected:**
- The conversation card appears with status `RECEIVED` and transitions to `SANITIZED` within ~1 s. The redacted opening message excerpt is visible; the raw text is not.
- Within ~10 s the triage block shows `category = SALES`, `confidence = high`, and a one-sentence reason. The status pill changes to `TRIAGED` then `ROUTED_SALES`.
- Within ~5 s of routing, the right column populates with the `SalesSpecialist` draft: a response text, an action chip (typically `INFO_PROVIDED` or `ORDER_PLACED`), and a `sales` specialist tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.

## J2 — Issues-and-repairs conversation: end-to-end handoff to IssuesRepairsSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop an issues-and-repairs-flavoured conversation (the seeded JSONL covers categories in rotation).

**Expected:**
- Triage emits `category = ISSUES_REPAIRS`. Status `ROUTED_ISSUES_REPAIRS`.
- `IssuesRepairsSpecialist` returns a `SpecialistResponse` with a concrete action (likely `REPLACEMENT_ARRANGED` or `REFUND_INITIATED`).
- Guardrail allows. Status `RESOLVED`.
- `SalesSpecialist` is never invoked for this conversation — no draft from it appears.

## J3 — Ambiguous message: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous conversation to drop.

**Expected:**
- Triage emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `ConversationEscalated`. Status `ESCALATED`.
- Neither specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the triage reason.

## J4 — Response guardrail blocks an out-of-policy draft

**Preconditions:** The seeded JSONL includes one conversation engineered to produce a next-day delivery promise from the specialist (and the mock-responses file has a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip conversation to drop and be triaged as `SALES`.
2. Wait for `SalesSpecialist` to produce a draft.

**Expected:**
- Status reaches `RESPONSE_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-delivery-window`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.

## J5 — Tool-call guardrail cancels an out-of-authority refund

**Preconditions:** The seeded JSONL includes one conversation engineered to produce a refund tool call that exceeds the policy ceiling.

**Steps:**
1. Wait for the trip conversation to drop and be triaged as `ISSUES_REPAIRS`.
2. Watch the specialist task loop in the right column.

**Expected:**
- The tool-call block appears in the right column showing `processRefund` with violation `refund-exceeds-ceiling`.
- Status briefly shows `TOOL_CALL_BLOCKED`.
- The specialist receives the rejection, uses its remaining iteration budget, and returns a `SpecialistResponse` with `action = ESCALATED` (or an alternative within authority).
- The conversation continues through the response guardrail and reaches `RESOLVED` or `BLOCKED` based on the specialist's alternative response.

## J6 — Operator unblocks a previously-blocked conversation

**Preconditions:** A conversation in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked conversation.
2. Enter a note ("operator override — reviewed with team lead").
3. Confirm.

**Expected:**
- A `POST /api/conversations/{id}/unblock` is sent.
- The conversation transitions to `RESOLVED`. The previously-blocked draft is now the published `SpecialistResponse`.
- The guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the operator's override and the `decidedBy` value.
