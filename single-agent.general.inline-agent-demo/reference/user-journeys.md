# User journeys — inline-agent-demo

## J1 — Submit a Q&A question with a seeded agent definition

**Preconditions:** Service running on declared port (`http://localhost:9801/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9801/` → App UI tab.
2. From the **Agent definition** dropdown, pick `Q&A assistant`.
3. The Agent definition textarea fills with the Q&A definition JSON. The Question field fills with the paired example question.
4. Click **Run**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `RUNNING` within 1 s as the workflow passes the definition to the agent.
- Within 30 s the card reaches `COMPLETED`. The right pane shows: the agent definition summary (agentName + truncated systemPrompt), the question, and a non-empty `answer` field with `tokenCount` and `answeredAt`.

## J2 — Malformed agent definition fails fast

**Preconditions:** Service running on declared port.

**Steps:**
1. In the **Agent definition** textarea, paste a JSON object with `agentName` present but `systemPrompt` absent (e.g., `{ "agentName": "test", "outputSchema": "...", "allowedTools": [] }`).
2. Type any question.
3. Click **Run**.

**Expected:**
- The service returns `400` and no card is added to the live list — the endpoint validates the request body before writing to the entity.
- If the validation check is in `validateStep` (definition reaches the entity but is caught by the workflow), the card appears briefly in `RECEIVED` then transitions immediately to `FAILED` with `failureReason` = "Agent definition is missing required field: systemPrompt". The right pane shows the failure reason in a red alert. No LLM call is made.

## J3 — Two concurrent runs with different agent definitions do not cross-contaminate

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a Q&A run (question: "Explain eventual consistency").
2. Without waiting for it to complete, immediately submit a keyword-extraction run (question: "Extract keywords from: cloud computing, distributed systems, latency").
3. Wait for both to reach `COMPLETED`.

**Expected:**
- The Q&A run's `response.answer` is a prose paragraph about eventual consistency. It contains no keyword list.
- The keyword-extraction run's `response.answer` is a keyword list (or JSON array). It contains no prose explanation of eventual consistency.
- Each run's agent name in the right pane matches its submitted `agentDefinition.agentName`.

## J4 — SSE stream delivers status updates without polling

**Preconditions:** Service running. Browser dev tools open to the Network panel.

**Steps:**
1. Open the `/api/runs/sse` stream in the Network panel (or observe via browser EventSource in the App UI tab).
2. Submit a new run.
3. Watch the SSE stream for events.

**Expected:**
- Within 1 s of each state transition (`RECEIVED`, `RUNNING`, `COMPLETED` or `FAILED`), one `run-update` event arrives on the stream.
- The event carries the full run row at the time of transition: `runId`, `status`, and all populated fields.
- The UI live list updates immediately on each event — no polling interval is visible in the Network panel.
- A late-joining client that connects to the SSE stream after the run completes receives the final `COMPLETED` event from the view's stream-update mechanism.

## J5 — Custom agent definition with a non-empty allowedTools list

**Preconditions:** Service running. The seeded tool list includes at least one named tool the runtime recognises (e.g., a no-op echo tool registered in the generated service).

**Steps:**
1. In the **Agent definition** textarea, paste a definition with `"allowedTools": ["echo"]` and a `systemPrompt` that instructs the agent to use the echo tool.
2. Submit a question.

**Expected:**
- The run completes without error. The `response.answer` reflects the agent having used the listed tool (or gracefully noting it returned no useful result for the given question).
- The service log shows the tool invocation request from the agent loop.
- No tool outside the `allowedTools` list was invoked.
