# User journeys — inbox-watcher

## J1 — Message arrives and gets drafted

**Preconditions:** Service running on declared port; valid model-provider API key set; `InboxPoller` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 15 s for the first simulated message.

**Expected:**
- Message appears with status RECEIVED, then within 1 s transitions to SANITIZED. The sanitized subject is visible; the raw subject is not.
- Within 1 s the classification chip appears (`AUTO_DRAFTED` for the simulator's seeded messages).
- Within 30 s the status reaches AWAITING_APPROVAL with a drafted reply visible in the detail pane.

## J2 — Approve a draft

**Preconditions:** A message in AWAITING_APPROVAL.

**Steps:**
1. Select the message in the list.
2. (Optionally) edit the draft body.
3. Click Approve.

**Expected:**
- Status transitions to SENT immediately. The detail pane shows `decision.approved = true` and the `decidedBy` field is set from the UI's session id.
- A `MessageSent` event is written to the entity; no actual outbound network call leaves the process (the `sendReply` tool is a no-op stub).
- The card moves to its terminal styling.

## J3 — Reject a draft

**Preconditions:** A message in AWAITING_APPROVAL.

**Steps:**
1. Click Reject. A small reason textarea opens.
2. Type a reason; click Confirm.

**Expected:**
- Status transitions to DISMISSED. `decision.approved = false`, `decision.reason` populated.
- The reason is visible on the card.

## J4 — Eval score appears

**Preconditions:** At least one SENT message exists with no `evalScore`. The `EvalRunner` schedule reduced for the test (`EVAL_RUNNER_SECONDS=60`).

**Steps:**
1. Approve a draft.
2. Wait 60 s.

**Expected:**
- The card shows an eval score chip (1–5) and the eval rationale is visible in the detail pane.
- The brief's row in `/api/inbox` includes `evalScore` populated.

## J5 — PII does not reach the LLM (audit check)

**Preconditions:** Service running with debug logging enabled (`LOG_LEVEL=DEBUG`).

**Steps:**
1. Submit a custom message via `POST /api/inbox` with body `"My email is test@example.com and my order is 12345"`.
2. Inspect the service log for the LLM call payload.

**Expected:**
- The log shows the LLM call body contains `[REDACTED-EMAIL]` and `[REDACTED-ORDER-ID]` — never `test@example.com` or `12345`.
- The entity's `incoming.body` (the audit log) still has the raw values; the entity's `sanitized.redactedBody` has the redacted form.
