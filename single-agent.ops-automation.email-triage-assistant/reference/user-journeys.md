# User journeys — email-triage-assistant

## J1 — Submit a billing-dispute email and get a draft

**Preconditions:** Service running on declared port (`http://localhost:9664/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9664/` → App UI tab.
2. From the **Seeded email** dropdown, pick `Billing Dispute`.
3. Click **Load seeded example** to fill the from address, subject, email body, and attachment text fields.
4. Click **Submit for triage**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted email body; PII category chips show at least `email`, `phone`, `person-name`.
- Within 30 s the card reaches `DRAFT_READY`. The right pane shows: an urgency badge (`HIGH`), a category chip (`BILLING`), the `classificationRationale` paragraph, and the full draft reply text.
- The **Approve** and **Reject** buttons appear in the Confirm section of the right pane. Both are enabled.

## J2 — User approves the draft and the email is sent

**Preconditions:** J1 completed; the email card is in `DRAFT_READY` state.

**Steps:**
1. Read the draft reply in the right pane.
2. Click **Approve**.

**Expected:**
- The UI immediately disables both buttons to prevent double-submission.
- The card transitions from `DRAFT_READY` to `SENDING` within 1 s of the approve POST landing in `ConfirmationEntity`.
- `SendGuardrail` confirms the `APPROVED` decision in `ConfirmationEntity` and permits the `send_email` tool call.
- Within 5 s the card reaches `SENT`. The right pane shows "Draft approved and sent." The service log contains one `send_email` tool execution entry for this `emailId`.
- No second `send_email` call appears in the log — the guardrail confirms approval once, and the tool executes once.

## J3 — User rejects the draft

**Preconditions:** J1 completed; the email card is in `DRAFT_READY` state.

**Steps:**
1. Read the draft reply in the right pane.
2. Click **Reject**.

**Expected:**
- Both buttons are disabled immediately.
- The card transitions from `DRAFT_READY` to `REJECTED` within 1 s.
- The right pane shows "Draft rejected. No email was sent." The status pill turns red.
- The service log contains no `send_email` tool execution entry for this `emailId`. The `SendGuardrail` was never reached because the workflow's `confirmStep` short-circuited to `recordRejected` on seeing the `REJECTED` decision.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit an email with `rawBody` containing the literal strings `jane.doe@example.com`, `+1 650 555 0199`, and `ACC-8820`, and an `attachments[0].textContent` containing `Card ending 4242`.
2. Wait for `DRAFT_READY`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/emails/{id}` and read `submission.rawBody` and `submission.attachments[0].textContent`.

**Expected:**
- The logged LLM call body (both `body.txt` and `attachments.txt` attachments) contains only redacted forms: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-ACCOUNT]`, `[REDACTED-PCN]`. The raw strings do not appear.
- `submission.rawBody` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists at minimum `email`, `phone`, `account-id`, `payment-card-number`.

## J5 — Guardrail blocks premature send in mock mode

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock `TriageResult` entries do not trigger `send_email` during `triageStep` — but a specially crafted mock entry can be configured to attempt the call before the user confirms.

**Steps:**
1. Submit any seeded email.
2. Before clicking **Approve** in the UI, observe the service log.

**Expected:**
- The service log shows no `send_email` tool execution for this email during `triageStep` or `confirmStep` (the workflow is paused at `confirmStep`).
- If a mock-LLM path attempts to invoke `send_email` before confirmation, the log shows one `guardrail.block` line with reason `send-pending-confirmation`. The tool call does not execute.
- After the user clicks **Approve**, the log shows one `send_email` tool execution entry — and zero `guardrail.block` entries from that point forward for this email.

## J6 — Attachment content influences classification

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a product-support email with a short body ("Please help, the app is broken.") and a `textContent` attachment that contains detailed error logs referencing a specific feature module.
2. Wait for `DRAFT_READY`.

**Expected:**
- The `classificationRationale` references the attachment content (e.g., "Attachment log indicates a failure in the payment module, confirming SUPPORT classification.").
- The category chip shows `SUPPORT`.
- The draft reply acknowledges the error detail from the attachment, not just the vague body text.
- The sanitized attachment preview in the right pane is visible and non-empty, confirming the attachment text was processed.
