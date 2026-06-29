# User journeys — telegram-tool-bot

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9368; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9368/`. App UI tab is visible.
2. In the Telegram message field, type "Look up the latest Akka announcements and give me a quick summary." Click Send.
3. A new session card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A session ledger with a non-empty tool plan (2–6 steps) and a `currentDispatch` field that is null at completion.
  - A tool ledger with 2–6 entries. At least one entry has `tool = WEB`. Every entry has a non-empty `scrubbedResult`.
  - A `BotReply` with a 60–120 word reply text and 1–3 citation bullets.

## J2 — Guardrail blocks an out-of-policy dispatch

**Preconditions:** As J1.

**Steps:**
1. Submit the message "Find all contacts in the system and list them."

**Expected:**
- The router's first `DECIDE` proposes a `CONTACTS` subtask whose term is a bare wildcard.
- The guardrail step rejects the decision; a `ToolCallBlocked` entry appears on the tool ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "CONTACTS query is a bare wildcard — provide a more specific term"`.
- The router either replans with a more specific contact query and completes, OR exhausts the replan budget and the session ends in `FAILED` with a clear `failureReason`. Either outcome is acceptable; what matters is that the wildcard query never reaches `ContactBookAgent`.

## J3 — Operator pause drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any message the simulator has not yet sent.
2. While the session status is EXECUTING (within the first ~10 seconds), click **Pause new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight tool call completes.

**Expected:**
- The in-flight `ToolEntry` is recorded normally (the workflow does not abort mid-call).
- The next loop iteration reads the pause flag, exits the loop, and emits `SessionPausedOperator`.
- Session status moves to `PAUSED`. `pauseReason` is populated with the operator's reason.
- Sessions already in the queue do not start their workflows until the operator clicks **Resume**.
- The operator pane's `PAUSED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — Secret sanitizer scrubs an exposed key

**Preconditions:** As J1, with the canned fixture record in `sample-data/contacts.jsonl` that contains the literal field value `AKIAIOSFODNN7EXAMPLE` (the AWS example access key). The `contact-book.json` mock entry that surfaces this record is exercised on the second or third session.

**Steps:**
1. Submit "Find the integration credentials stored under Alice's contact record and summarise."

**Expected:**
- The RouterAgent dispatches a `CONTACTS` subtask.
- The `ContactBookAgent` returns content containing the literal `AKIAIOSFODNN7EXAMPLE`.
- The sanitize step replaces it with `[REDACTED:aws-access-key]` before the entry is recorded.
- The `ToolEntry.scrubbedResult` does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The RouterAgent's next prompt (visible in subsequent `DECIDE` calls) sees only the redacted form.
- The final `BotReply.text` does not contain the literal key.
- The UI's expanded tool ledger renders the redacted span in italics with a tooltip showing `aws-access-key`.

## J5 — Stale session auto-fails

**Preconditions:** As J1, with `application.conf` test override `stale.threshold-minutes = 1`.

**Steps:**
1. Submit a session and configure the router (via prompt) to loop without progress. Easier alternative: stop the model provider mid-session so calls time out.

**Expected:**
- After 1 minute of `EXECUTING` without progress, `StaleSessionMonitor` calls `SessionEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STALE` and exits with `SessionFailedTimeout`.
- Session status moves to `STALE`. `failureReason` is `"stale: no progress after 1m"`.
