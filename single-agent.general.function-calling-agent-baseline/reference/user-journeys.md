# User journeys — function-calling-agent-baseline

Acceptance bar for the generated system. Each journey has preconditions, numbered steps, and expected outcomes. A green check against all journeys means the blueprint is working as specified.

---

## J1 — Multi-step arithmetic query (happy path)

**Preconditions:** Service is running. All three tools (calculator, dictionary, date-formatter) are enabled. Mock LLM or a real model is configured.

**Steps:**

1. Open the App UI tab.
2. Select the seeded arithmetic query: "If a train travels at 60 km/h for 2.5 hours and then at 80 km/h for 1.5 hours, what is the total distance?"
3. Verify all three tools are checked in the Tool set panel.
4. Click **Run query**.
5. Observe the run card appear in the live list in `SUBMITTED` state.
6. Within 2 s, the card transitions to `RUNNING`.
7. Within 30 s, the card transitions to `ANSWER_RECORDED`.
8. Open the right-pane detail for this run.

**Expected:**
- The answer field contains the correct result (270 km).
- The tool-call trace shows at least two `calculator` invocations.
- The guardrail summary shows `toolCallBlocked: false`, `answerBlocked: false`.
- No `FAILED` state is shown at any point.

---

## J2 — Before-tool-call guardrail blocks an invalid invocation

**Preconditions:** Service is running. Mock LLM path is active. The seed response file contains one entry whose first tool call names a tool not in the enabled list (e.g., `web-search`).

**Steps:**

1. Submit a query with only the `calculator` tool enabled.
2. The mock LLM on the first iteration proposes a call to `web-search`.
3. Observe the run card transitions to `RUNNING`.
4. Within 30 s, the card transitions to `ANSWER_RECORDED`.
5. Open the right-pane detail.

**Expected:**
- The tool-call trace shows the blocked call: `toolName: "web-search"`, `blocked: true`.
- A subsequent iteration shows a valid `calculator` call that succeeded.
- The guardrail summary shows `toolCallBlocked: true`, `toolCallBlockCount: 1`.
- The final answer is well-formed and non-empty.
- The UI displays the `BLOCKED_TOOL_CALL` badge on the run card.

---

## J3 — Before-agent-response guardrail blocks a credential-containing answer

**Preconditions:** Service is running. Mock LLM path is active. The seed response file contains one entry whose first proposed `AgentAnswer.answer` includes a credential-like token (e.g., `Bearer eyJhbGciO...`).

**Steps:**

1. Submit any query. The mock LLM on the first final-answer attempt returns the seeded credential-containing answer.
2. Observe the run card in `RUNNING`.
3. Within 30 s, the card transitions to `ANSWER_RECORDED`.
4. Open the right-pane detail.

**Expected:**
- The final displayed answer does not contain the credential-like token.
- The guardrail summary shows `answerBlocked: true`, `answerBlockCount: 1`.
- The UI displays the `BLOCKED_ANSWER` badge on the run card.
- The entity log (via `GET /api/runs/{id}`) confirms that only the cleaned answer is stored in `answer.answer`.

---

## J4 — Direct answer without any tool calls

**Preconditions:** Service is running. Mock LLM path is active with a seed entry that returns an `AgentAnswer` with an empty `toolCallTrace`.

**Steps:**

1. Submit a query with no tools enabled (uncheck all checkboxes).
2. Click **Run query**.
3. Observe the card transitions: SUBMITTED → RUNNING → ANSWER_RECORDED.
4. Open the right-pane detail.

**Expected:**
- The tool-call trace section is empty (no rows).
- The `totalIterations` field is 1.
- The answer is non-empty.
- The guardrail summary shows both block flags false.
- No `FAILED` state was reached despite no tool calls being available.

---

## J5 — Run failure on iteration budget exhaustion

**Preconditions:** Service is running. Mock LLM path is active. A seed response is configured such that every proposed answer fails the `AnswerGuardrail` check for 6 consecutive iterations.

**Steps:**

1. Submit any query.
2. Observe: the run card stays in `RUNNING` for up to 120 s (the `runStep` timeout).
3. After the budget is exhausted, observe the card transition to `FAILED`.

**Expected:**
- The final status is `FAILED`.
- `GET /api/runs/{id}` returns `status: "FAILED"` and `answer: null`.
- The UI shows the `FAILED` status pill in red.
- No partial or malformed answer is stored in the entity log.

---

## J6 — SSE stream delivers state transitions in order

**Preconditions:** Service is running. A client opens `GET /api/runs/sse` before submitting a run.

**Steps:**

1. Open the SSE stream in a browser or via `curl -N /api/runs/sse`.
2. Submit a query in the App UI.
3. Observe the SSE events arriving.

**Expected:**
- The stream delivers events in status order: `SUBMITTED`, `RUNNING`, `ANSWER_RECORDED` (or `FAILED`).
- Each event carries the full `AgentRun` row at the time of transition (no partial objects).
- A client that connects after the run completes and calls `GET /api/runs` sees the final state without needing to replay SSE events.
