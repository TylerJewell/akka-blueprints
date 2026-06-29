# User journeys

Each journey is an acceptance test for the generated system.

## J1 — Ingest, sanitize, and draft

**Preconditions:** service running on `http://localhost:9165/`; a model provider configured (real or mock).

**Steps:**
1. POST `/api/emails` with a body carrying a phone number and an email address, or wait for `InboxMonitor` to drip one.
2. Subscribe to `/api/emails/sse` in the App UI tab.

**Expected:** the email appears in `RECEIVED`, then `SANITIZED` with the phone number and email address masked in `sanitizedBody`, then `DRAFTED` with a non-empty `draftBody` — typically within 30s.

## J2 — Sensitive reply gated by a human

**Preconditions:** an email whose content concerns a refund, complaint, account, or legal matter.

**Steps:**
1. Ingest the sensitive email.
2. Watch it reach `DRAFTED` then `AWAITING_REVIEW`.
3. Click Approve (POST `/api/emails/{id}/approve`).

**Expected:** the email pauses in `AWAITING_REVIEW` with Approve/Reject buttons; after Approve it transitions to `SENT` with a non-null `sentMessageId`; the buttons disappear.

## J3 — Routine reply auto-sends

**Preconditions:** an email with a routine question (order status, hours).

**Steps:**
1. Ingest the routine email.
2. Take no action.

**Expected:** the email moves `RECEIVED → SANITIZED → DRAFTED → SENT` without a human step; no Approve/Reject buttons ever appear.

## J4 — Reject a sensitive draft

**Preconditions:** a sensitive email in `AWAITING_REVIEW`.

**Steps:**
1. Click Reject and enter a reason (POST `/api/emails/{id}/reject`).

**Expected:** the email transitions to `REJECTED`; the UI shows the reject reason; the reply is never sent.

## J5 — Escalation on review timeout

**Preconditions:** a sensitive email in `AWAITING_REVIEW`, untouched.

**Steps:**
1. Leave it for more than two minutes.

**Expected:** `StuckReviewMonitor` transitions it to `ESCALATED`; the Approve/Reject buttons disappear.

## J6 — Background load from the monitor

**Preconditions:** service running, no UI interaction.

**Steps:**
1. Observe the App UI list.

**Expected:** `InboxMonitor` seeds emails from `sample-events/inbound-emails.jsonl` every 30s; each starts a fresh `ResponderWorkflow` and appears in the live list.
