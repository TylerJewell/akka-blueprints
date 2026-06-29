# User journeys — guided-intake-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: goal met within turn budget

**Preconditions:** Service running on port 9314; valid model-provider API key set (or `model-provider = mock`). The "support-ticket-intake" goal is present in `GoalCatalogEntity` (seeded from `sample-data/goals.json`).

**Steps:**
1. Open `http://localhost:9314/`. App UI tab is visible.
2. In the session form, select "Support Ticket Intake" from the goal selector. Click **Start session**.
3. A new conversation card appears with status PLANNING.
4. Within 5 s, a `question-ready` SSE event arrives; the active conversation pane shows the first question.
5. Type a reply and click **Send reply**. Repeat for each question the agent asks.

**Expected:**
- After 3–6 turns, the agent emits `GoalMet`; status transitions to COMPLETED.
- The expanded view shows:
  - An `IntakePlan` with an ordered question sequence matching the goal's question set.
  - A turn log with 3–6 `TurnRecord` entries, all with `verdict = OK`.
  - An `IntakeSummary` with all required fields populated (`product`, `stepsToReproduce`, `severity`) and `confidence >= 0.8`.
- Every turn transition is visible in real time via the `conversation-update` SSE event.

## J2 — Guardrail blocks an off-topic proposed question

**Preconditions:** As J1. Configure the mock LLM (or use a prompt-engineered fixture) so that the second `PROPOSE_QUESTION` call returns a question asking for the user's bank account number.

**Steps:**
1. Start a "support-ticket-intake" session.
2. Answer the first question normally.
3. Observe the second question proposed by the agent.

**Expected:**
- The guardrail step rejects the proposal; a `QuestionBlocked` entry appears in the turn log with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker` containing "financial account number".
- The agent is asked to revise; the next proposed question stays within the goal's support domain.
- The blocked question text never appears in the active conversation pane or in any SSE `question-ready` event.

## J3 — PII sanitizer scrubs a reply containing personal identifiers

**Preconditions:** As J1.

**Steps:**
1. Start a "support-ticket-intake" session.
2. When prompted for a description, type: "My name is Jane Doe, you can reach me at jane@example.com or 555-010-0100. The problem is the export page freezes."

**Expected:**
- The `TurnRecord.sanitizedReply` reads: "My name is [REDACTED:name], you can reach me at [REDACTED:email] or [REDACTED:phone]. The problem is the export page freezes."
- The raw reply ("Jane Doe", "jane@example.com", "555-010-0100") does not appear in any event, view row, or SSE payload.
- The agent's next proposed question does not reference the original name, email, or phone number.
- The UI's turn log renders the redacted spans in italics with tooltips showing the tag type.

## J4 — Operator halt drains the in-flight turn gracefully

**Preconditions:** As J1.

**Steps:**
1. Start any intake session.
2. While the session status is ELICITING, click **Halt new sessions** in the operator pane and provide a reason.
3. Submit a reply to the current open question.

**Expected:**
- The in-flight reply is sanitized and recorded normally; the corresponding `TurnRecord` event is emitted.
- The workflow's next loop iteration reads the halt flag, exits the loop, and emits `ConversationHaltedOperator`.
- Conversation status moves to `HALTED`. `haltReason` is populated with the operator's reason text.
- No further `question-ready` SSE events are emitted for this session.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- Clicking **Resume** clears the halt flag; new sessions can be started again.

## J5 — Stale session is marked ABANDONED

**Preconditions:** As J1, with `application.conf` test override `stale.threshold-minutes = 1`.

**Steps:**
1. Start a session.
2. Do not submit any reply. Wait 1 minute.

**Expected:**
- `StaleSessionMonitor` calls `ConversationEntity.abandonConversation`.
- The workflow's next `evaluateStep` reads `status = ABANDONED` and exits.
- Conversation status moves to `ABANDONED`. `failureReason` is `"abandoned: no reply after 1m"`.
- The session card in the UI updates to reflect `ABANDONED` status via SSE.
