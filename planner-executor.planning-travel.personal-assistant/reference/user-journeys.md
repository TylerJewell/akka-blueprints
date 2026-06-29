# User journeys — line-personal-assistant

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Create a calendar event (happy path)

**Preconditions:** Service running on port 9573; valid model-provider API key set (or `model-provider = mock`); Google Calendar OAuth token set (or mock calendar client active).

**Steps:**
1. Open `http://localhost:9573/`. App UI tab is visible.
2. In the simulated chat input, type "Schedule a team sync with Alice tomorrow at 3 PM for 1 hour." and press Send.
3. A new conversation card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- `PlannerAgent` returns `ActionPlan { intent: CREATE_EVENT, calendarParams: { action: CREATE, title: "Team sync with Alice", startTime: <tomorrow 15:00 UTC>, endTime: <tomorrow 16:00 UTC> } }`.
- The guardrail passes (startTime and endTime are present; intent is valid).
- `CalendarExecutorAgent` is called; `ExecutionResult.ok = true`.
- Status transitions to COMPLETED.
- The expanded conversation shows the plan intent chip, the calendar params, and the execution summary containing the event title and time.
- The simulated LINE chat pane shows the assistant's reply: "Created event: 'Team sync with Alice' on …".

## J2 — Send an email with user confirmation

**Preconditions:** As J1; Gmail OAuth token set (or mock Gmail client active).

**Steps:**
1. Type "Email the project update to bob@example.com" in the chat input and press Send.
2. A conversation card appears with status PLANNING, then transitions to AWAITING_CONFIRMATION.
3. The confirmation gate in the expanded card shows the recipient, subject, and Approve / Deny buttons.
4. Click **Approve**.

**Expected:**
- The guardrail passes (recipientEmail is present; `example.com` is not on the deny-list; subject is inferable).
- The workflow sends a LINE confirmation message and parks in `AWAITING_CONFIRMATION`.
- `ConfirmationEntity` status is `PENDING`.
- After clicking Approve, `POST /api/confirmations/{id}/reply { approved: true }` is called.
- `ConfirmationEntity` status transitions to `APPROVED`.
- The workflow resumes; `GmailExecutorAgent` is called; `ExecutionResult.ok = true`.
- Conversation status transitions to COMPLETED.
- The simulated chat pane shows "Email sent to bob@example.com. Subject: Project update."

## J3 — Guardrail blocks a deny-listed recipient

**Preconditions:** As J1; `akka.samples.email-deny-domains = ["rival.io"]` in `application.conf`.

**Steps:**
1. Type "Email the project report to attacker@rival.io" and press Send.

**Expected:**
- `PlannerAgent` produces `ActionPlan { intent: SEND_EMAIL, gmailParams: { recipientEmail: "attacker@rival.io", … } }`.
- The guardrail detects that `rival.io` is on the deny-list and rejects the plan.
- The workflow emits `ActionBlocked`; conversation status transitions to BLOCKED.
- No `ConfirmationEntity` is created; `GmailExecutorAgent` is never called.
- The expanded conversation card shows the blockerReason: "Email recipient domain 'rival.io' is not permitted."
- The simulated chat pane shows the assistant's explanation message.

## J4 — Confirmation window expires

**Preconditions:** As J2. For testing, reduce `akka.samples.confirmation-timeout-minutes = 1` in `application.conf`.

**Steps:**
1. Type "Email Alice the agenda" and press Send.
2. The planner asks for Alice's email (CLARIFY intent, or if email is supplied, moves to AWAITING_CONFIRMATION).
3. Assume AWAITING_CONFIRMATION is reached. Do NOT click Approve or Deny.
4. Wait for `StaleConversationMonitor` to tick (up to 2 minutes with the reduced timeout).

**Expected:**
- After 1 minute in `AWAITING_CONFIRMATION`, `StaleConversationMonitor` calls `ConversationEntity.expireConversation`.
- `ConfirmationEntity` status transitions to `EXPIRED`.
- Conversation status transitions to `EXPIRED`.
- `GmailExecutorAgent` is never called.
- The conversation card shows the EXPIRED status pill.
- The simulated chat pane shows a timeout notice: "Your email request expired before confirmation."
