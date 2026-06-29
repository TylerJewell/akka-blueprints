# User journeys — web-navigation-agent

## J1 — Submit a task and reach COMPLETED

**Preconditions:** Service running on declared port (`http://localhost:9797/`); a valid model-provider API key set (vision-capable model), or the mock LLM selected at scaffold time. The allowed-domain list includes `doc.akka.io`.

**Steps:**
1. Open `http://localhost:9797/` → App UI tab.
2. In the **Task goal** field, enter `Find the latest Akka SDK release version on doc.akka.io`.
3. In **Starting URL**, enter `https://doc.akka.io`. Leave **Max steps** at 20.
4. Click **Start navigation**.

**Expected:**
- The new session card appears in the live list with status `STARTING` within 1 s, then transitions to `NAVIGATING` within 2 s.
- The right pane shows the starting URL and the first screenshot thumbnail.
- On each action step the action log table gains a new row (action type, target, rationale, timestamp). The screenshot thumbnail updates after each executed action.
- Within the step budget the card transitions to `COMPLETED`. The right pane shows a green outcome badge, the `taskResult` string (the release version found), and the steps used.
- No `AWAITING_APPROVAL` state occurs on this task because no high-stakes actions are needed to browse documentation.

## J2 — Operator halt stops an active session

**Preconditions:** Service running. Any model provider. A session is in `NAVIGATING` state (J1, paused mid-way by extending `maxSteps` to 50 to give time to act).

**Steps:**
1. While the session card shows `NAVIGATING`, click the red **Halt all sessions** button in the UI.
2. Observe the session card.

**Expected:**
- Within one action step (at most one more agent call completes in-flight; no new ones start), the session card transitions to `HALTED`.
- The action log shows the last executed action; no new rows appear after the halt.
- The service log shows a `halt.check: halted=true` line followed by `session.halted` for the session id.
- A second open session (if any) also stops before its next action step.
- Clicking **Clear halt** in the UI calls `DELETE /api/halt`. The halt banner disappears. New sessions can be started; existing halted sessions remain halted (the halt did not undo already-halted sessions).

## J3 — HITL gate fires on a high-stakes action; human approves; session completes

**Preconditions:** Service running with the mock LLM selected. The mock `plan-next-action.json` includes an entry with `highStakes=true` that the seed selector triggers on step 3 of every second session (deterministic via `seedFor`).

**Steps:**
1. Submit any seeded task. Watch the session card.
2. When the card transitions to `AWAITING_APPROVAL`, observe the right pane: the proposed action (type, target, rationale) and the orange-bordered Approve / Reject panel are visible.
3. Click **Approve**.

**Expected:**
- The session transitions from `NAVIGATING` to `AWAITING_APPROVAL` when the guardrail flags the high-stakes action.
- The action log shows the pending action row in an `awaiting` state — not `executed`.
- On **Approve**, the session transitions back to `NAVIGATING`. The action log marks the action as `executed` and the screenshot thumbnail updates.
- Navigation continues and eventually the session reaches `COMPLETED` (or the step budget is exhausted).
- The `HitlResolved{outcome: APPROVED}` event is visible in the service log.

## J4 — HITL rejection causes agent to replan

**Preconditions:** Same as J3. The mock includes a `highStakes=true` action entry.

**Steps:**
1. Submit any seeded task. Wait for `AWAITING_APPROVAL`.
2. Click **Reject** (fills reason: "Action not authorised").

**Expected:**
- The session transitions back to `NAVIGATING` (not to `REJECTED` — `REJECTED` is a terminal state; this is a mid-session rejection of one specific action).
- The action log shows the rejected action row with a `rejected` badge and the reason.
- The agent receives a structured rejection message and proposes a different action on the next step. The action log grows with the new proposal.
- If the agent's next proposal is also high-stakes, another HITL gate fires. If the agent's next proposal is not high-stakes and passes the guardrail, it executes immediately.

## J5 — Guardrail blocks navigation to a disallowed domain

**Preconditions:** Service running with the mock LLM. The allowed-domain list does NOT include `evil.example.com`. The mock `plan-next-action.json` includes one entry with `targetUrl = "https://evil.example.com/page"`.

**Steps:**
1. Submit any seeded task whose `seedFor` deterministically selects the disallowed-domain mock entry on step 1.
2. Observe the action log.

**Expected:**
- Step 1's action row shows `executed: false` with `blockedReason: "domain-not-allowed: evil.example.com"`.
- The session does NOT transition to `FAILED` — the block is recoverable. The agent receives a `block` rejection and retries on step 2 with a different action.
- Step 2's action row shows a different target URL (within the allowed-domain list) and `executed: true`.
- The service log shows a `guardrail.block` line for step 1 naming the blocked domain.

## J6 — Session exceeds step budget and terminates with partial result

**Preconditions:** Service running. Any model provider. `maxSteps` set to 3 via the UI.

**Steps:**
1. Submit a task with a complex goal that is unlikely to complete in 3 steps.
2. Wait for the session to exhaust its step budget.

**Expected:**
- After 3 action steps the workflow's `completeStep` fires automatically with `success: false` and a `taskResult` describing the partial progress made.
- The session card transitions to `COMPLETED` (not `FAILED`). The outcome badge is amber.
- The action log has exactly 3 rows (or fewer, if some were blocked and counted separately).
- The right pane shows `stepsUsed: 3` and the partial `taskResult` from the agent's last state.
