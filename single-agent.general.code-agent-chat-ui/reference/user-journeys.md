# User journeys — code-agent-chat-ui

## J1 — Ask a question and receive a streamed answer

**Preconditions:** Service running on declared port (`http://localhost:9809/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9809/` → App UI tab.
2. Click the seeded chip **"What is the current Python version?"**.
3. Observe the new session appear in the left rail with status `ACTIVE`.
4. Watch the right-pane chat thread.

**Expected:**
- A USER bubble appears immediately with the question text.
- Within 5 s, a collapsible **Steps** panel appears below an in-progress ASSISTANT bubble. It shows a `web-search` tool-call entry in `APPROVED` state.
- Within 15 s, a `code-execution` tool-call entry appears in `APPROVED` state in the same Steps panel.
- Within 30 s, the ASSISTANT bubble's content completes. It contains at least one sentence of explanation and one code block. The session transitions to `IDLE`.
- The left-rail session item shows `IDLE` status and an updated age.

## J2 — Before-tool-call guardrail blocks a disallowed code call

**Preconditions:** Service running with the mock LLM (`model-provider = mock`). The mock's `answer-question.json` includes an entry whose first tool call attempts `import subprocess; subprocess.run([...])`.

**Steps:**
1. Open a new session.
2. Type `Show me how to list the root filesystem` and click **Send**.

**Expected:**
- The Steps panel shows a `code-execution` tool call with status `BLOCKED` and block reason `code-execution-policy: subprocess spawn disallowed`.
- No code execution actually runs. The `ToolCallResolved` event on the entity records `BLOCKED`.
- The agent's subsequent iteration does not include the blocked call. It produces a response that acknowledges the sandbox constraint and offers an alternative (e.g., describing what the command would return).
- The final ASSISTANT bubble does not contain raw subprocess output.

## J3 — Before-agent-response guardrail rejects a malformed response

**Preconditions:** Mock LLM mode. The mock's second malformed entry returns a bare JSON object as `responseText`.

**Steps:**
1. Submit any seeded question three times in a row.
2. Watch the second session's agent lifecycle in the network panel (`/api/sessions/{id}/sse`).

**Expected:**
- The agent's first iteration on one of those sessions produces a `responseText` that matches `^\s*\{` (raw JSON dump).
- The `before-agent-response` guardrail rejects it. The malformed text NEVER appears in the UI — there is no `AssistantMessageRecorded` event with the raw JSON payload.
- The agent loop retries on iteration 2. The second iteration produces a human-readable markdown response that passes all four guardrail checks.
- The service log shows one `guardrail.reject` line for the rejected iteration, naming the `raw-json-response` check.

## J4 — Plan revision badge appears at step 3

**Preconditions:** Service running (real or mock LLM). A question that causes the agent to make at least 3 tool calls (e.g., "Search for event sourcing patterns, write a Python example, and explain the output").

**Steps:**
1. Open a new session.
2. Send the multi-step question.
3. Watch the Steps panel in the ASSISTANT bubble.

**Expected:**
- The Steps panel lists tool calls in order: first a `web-search`, then a `code-execution`.
- After step 3 (the third tool call), a `[Replanned at step 3] <revised goal>` entry appears in muted italic between the third and fourth tool-call rows.
- A `PlanRevised` event is recorded on `ChatSessionEntity`.
- The UI renders the plan-revision badge without reloading the page (SSE-driven update).

## J5 — Session history survives an idle/active cycle

**Preconditions:** Service running. A session with at least 2 user/assistant turn pairs that has transitioned to IDLE.

**Steps:**
1. Start a session, send two questions, wait for the session to reach `IDLE` (30 s inactivity).
2. Send a third question in the same session.

**Expected:**
- The workflow re-activates the session: `SessionStatus` transitions `IDLE → ACTIVE`.
- The agent receives the full prior conversation history (both earlier turns) in its task instructions.
- The ASSISTANT response to the third question demonstrates awareness of the earlier turns (either by referencing them or by not re-explaining context that was already established).
- The session's `messages` list in `GET /api/sessions/{id}` contains all turns from all three exchanges.

## J6 — Tool-call audit log is complete regardless of outcome

**Preconditions:** Service running. Any session where at least one tool call was blocked.

**Steps:**
1. Trigger J2 (blocked code call).
2. Wait for the session to reach `IDLE`.
3. Fetch `GET /api/sessions/{id}` and inspect the `messages[].toolCalls` array on the ASSISTANT message.

**Expected:**
- The `toolCalls` array contains one entry with `status: BLOCKED` and a non-null `blockReason`.
- The entry has a `requestedAt` timestamp and a `resolvedAt` timestamp (both set even for blocked calls).
- No `outputSummary` is present for the blocked entry (`null` on the wire).
- The subsequent tool calls (if any, from the agent's retry iteration) are also present as separate entries in the array, each with their own status.
