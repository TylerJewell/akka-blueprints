# User journeys — realtime-conversational-agent

## J1 — Start a session and complete a 3-turn conversation

**Preconditions:** Service running on `http://localhost:9397/`; a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9397/` → App UI tab.
2. Enter `customer-demo-01` as the Customer ID, select `retail-support` as the persona, and click **Start session**.
3. Wait for the greeting to appear in the conversation thread.
4. Type "What are your store hours?" and click **Send**.
5. Wait for the agent reply.
6. Type "Do you offer free returns?" and click **Send**.
7. Wait for the agent reply.
8. Type "Thanks, that's all." and click **Send**.
9. Wait for the agent reply.
10. Click **End session**.

**Expected:**
- The new session card appears in the left rail with status `GREETING` within 1 s.
- The card transitions to `ACTIVE` and the greeting bubble appears in the right pane within 5 s.
- Each customer message produces an agent reply within 30 s. The reply bubbles appear on the right side; customer bubbles on the left.
- After **End session**, the status transitions to `SUMMARIZING` then `CLOSED` within 2 s.
- The session summary section appears at the bottom of the right pane with a quality score chip (4–5 expected for a clean session), a summary paragraph, `totalTurns = 4` (including the greeting), and `guardrailEvents = 0`.

## J2 — Guardrail blocks a policy-violating reply

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `reply-to-customer.json` includes deliberately violating entries (one with profanity; one referencing a competitor brand by name).

**Steps:**
1. Start a session as in J1 step 2.
2. Send four messages in sequence (any text). The fourth message will trigger the mock's violating path on the first iteration (modulo seed).
3. Watch the network panel of the browser dev tools (`/api/sessions/sse`) during the fourth turn.

**Expected:**
- The fourth turn's first agent iteration produces a policy-violating candidate reply.
- The `before-agent-response` guardrail rejects it. The violating text NEVER appears in `SessionEntity` — there is no `AgentReplied` event with the violating payload.
- The agent loop retries on iteration 2 and produces a passing reply. The reply bubble appears in the thread with a yellow `⚠ guardrail` badge indicating the guardrail was triggered during this turn.
- The service log shows one `guardrail.reject` line naming the violated category and the offending fragment.

## J3 — Guardrail exhaustion produces a failed turn and a low quality score

**Preconditions:** Mock LLM mode. Configured so that all three iterations of one turn return policy-violating replies (this requires a test override in `MockModelProvider` where `seedFor(sessionId, turnIndex) % 10 < 3` selects a violating entry on each of the 3 iterations).

**Steps:**
1. Start a session.
2. Wait for the greeting.
3. Send a message that triggers the all-iterations-violating path.

**Expected:**
- All three iterations are rejected by the guardrail.
- The workflow's `converseTurnStep` fails over to `error`. `SessionEntity` records a `TurnFailed` event for that turn.
- The turn bubble in the conversation thread shows a red error indicator: "Agent could not produce a safe reply."
- The session card status transitions to `FAILED`.
- If the operator clicks **End session** (which closes the session even when failed), the `TurnSummarizer` produces a quality score of 1 with a rationale noting the guardrail-exhaustion event.
- The session card border highlights red.

## J4 — Late-joining browser receives full session history via SSE

**Preconditions:** Service running. An existing open session with at least 2 completed turns.

**Steps:**
1. Note the `sessionId` of an active session.
2. Open a second browser tab and navigate to `http://localhost:9397/`.
3. Find the same session in the left rail and click it.

**Expected:**
- The right pane immediately shows all prior turns (customer + agent bubbles) from the current view state — no additional REST call or page reload required.
- The SSE connection for the new tab delivers the full session row (including all prior turns) on the initial connection event.
- Subsequent turns sent from the first tab appear in both tabs within 1–2 s.

## J5 — Closed session has consistent data

**Preconditions:** Service running. Complete a full session with 3 customer turns (J1 steps 1–10).

**Steps:**
1. After the session reaches `CLOSED`, fetch `GET /api/sessions/{sessionId}` from the browser dev tools or curl.

**Expected:**
- `status` is `"CLOSED"`.
- `turns` has exactly 4 entries (1 greeting + 3 customer turns), all with `status = "AGENT_REPLIED"`.
- Each `turns[i].agent` is non-null and `replyText` is non-empty.
- `summary` is non-null. `summary.totalTurns` equals 4. `summary.guardrailEvents` equals 0 for a clean session (J1 path).
- `closedAt` is set and is after `openedAt`.
