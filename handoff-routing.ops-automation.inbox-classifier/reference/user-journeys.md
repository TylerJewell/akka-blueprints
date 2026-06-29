# User journeys — inbox-classifier

## J1 — Urgent message: end-to-end routing to FLAG_URGENT

**Preconditions:** Service running on declared port; valid model-provider key set (or mock LLM); `InboxSimulator` enabled.

**Steps:**
1. Open `http://localhost:9159/` → App UI tab.
2. Wait up to 30 s for the first simulated message seeded as urgent-flavoured to drop.

**Expected:**
- The message appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; any raw email addresses in the body are not.
- Within ~10 s the classification block shows `label = URGENT`, `confidence = high`, and a one-sentence reason. The status pill changes to `CLASSIFIED` then `ROUTING_DECIDED`.
- The routing decision shows `action = FLAG_URGENT` with a one-sentence reason. No guardrail block appears (non-destructive action).
- The status pill transitions to `ACTIONED`. `finishedAt` is set.
- Within ~10 s of the classification, the classification score chip shows a number 1–5 with a rationale.

## J2 — Spam message: MOVE_TO_SPAM passes the guardrail

**Preconditions:** Same as J1. The seeded JSONL includes a spam-flavoured message.

**Steps:**
1. Wait for the simulator to drop a spam-flavoured message.

**Expected:**
- Classification emits `label = SPAM`. Status `ROUTING_DECIDED`.
- Routing agent returns `action = MOVE_TO_SPAM`.
- The guardrail is invoked (destructive action). The guardrail block in the UI shows a green "allowed" check.
- Status transitions to `ACTIONED`.
- `ClassifierAgent` was the only agent to see the sanitized body; `RoutingAgent` received the same sanitized content.

## J3 — Info message: files to folder, no guardrail

**Preconditions:** The seeded JSONL includes an order-confirmation or newsletter message.

**Steps:**
1. Wait for an info-flavoured message to drop.

**Expected:**
- Classification emits `label = INFO` with `confidence = high`.
- Routing agent returns `action = MOVE_TO_FOLDER` with `targetFolder = "Archive"` (or `MARK_READ`). No guardrail is invoked.
- Status transitions to `ACTIONED` directly from `ROUTING_DECIDED`.
- Neither `MOVE_TO_SPAM` nor `DELETE` was attempted; the guardrail step was skipped entirely.

## J4 — Guardrail blocks an unwarranted DELETE

**Preconditions:** The seeded JSONL includes one message engineered to coax a `DELETE` action from the routing agent (and the mock-responses file includes a matching guardrail-trip entry).

**Steps:**
1. Wait for the trip message to drop and be classified.
2. Wait for `RoutingAgent` to return `action = DELETE`.

**Expected:**
- Status reaches `ROUTING_DECIDED` then `GUARDRAIL_PENDING`.
- The guardrail block in the UI shows a red badge with the violation `unwarranted-delete` (or `low-confidence-destructive-action`).
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked routing decision (not executed) and an Unblock button.

## J5 — Operator unblocks a previously-blocked message

**Preconditions:** A message in `BLOCKED` (from J4 or a live incident).

**Steps:**
1. Click the Unblock button on the blocked message.
2. Enter a note ("operator override — manually verified as spam").
3. Confirm.

**Expected:**
- A `POST /api/messages/{id}/unblock` is sent.
- The message transitions to `ACTIONED`. The previously-blocked routing action is now the executed action.
- The guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the operator's override with `decidedBy` and the note text.

## J6 — Classification score appears on every classified message

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new message through the classification step.

**Expected:**
- Within ~10 s of `MessageClassified`, the per-message classification score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never affects the workflow — a low score does not block the routing action from executing.
