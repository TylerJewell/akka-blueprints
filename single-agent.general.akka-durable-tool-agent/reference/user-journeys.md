# User journeys — durable-weather-agent

## J1 — Submit a Berlin query via trigger_agent

**Preconditions:** Service running on `http://localhost:9227/`; a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9227/` → App UI tab.
2. Click the **Berlin** seeded city button to fill the query textarea.
3. Confirm invocation mode is `trigger_agent` (default).
4. Click **Submit query**.

**Expected:**
- The new card appears in the live list with status `QUEUED` within 1 s.
- The card transitions to `RUNNING` within 1 s.
- Within 30 s the card reaches `COMPLETED`. The right pane shows: a conditions card with temperature and condition label, a 3-day forecast table, an empty alerts list, a narrative summary of 2–4 sentences, and a tool-call evidence table with `EXECUTED` rows for `get_current_weather` and `get_forecast`.
- The mode badge on the card reads `TRIGGER`.

## J2 — Invoke via call_agent (child-workflow composition)

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI, pick the **Tokyo** seeded city button.
2. Switch invocation mode to `call_agent`.
3. Click **Submit query**.

**Expected:**
- A new job card appears for the child workflow with mode badge `CALL` and status `QUEUED → RUNNING → COMPLETED`.
- The child job's detail pane shows the same conditions / forecast / tool-call structure as J1.
- If a parent `jobId` is present in the entity (i.e., the parent workflow started the child via `asyncCall`), the CALL badge links to the parent card.
- The child job completes independently; the parent's step that called `asyncCall(AgentWorkflow::callAgent, childJobId)` resumes after the child's `done` transition fires.

## J3 — Guardrail blocks an out-of-range forecast horizon

**Preconditions:** Service running. Mock LLM mode or real LLM. The seeded query for this journey asks for a 30-day forecast.

**Steps:**
1. In the query textarea, type: "30-day forecast for São Paulo".
2. Leave mode as `trigger_agent`.
3. Click **Submit query**.

**Expected:**
- The job reaches `RUNNING`.
- In the tool-call evidence table, a `BLOCKED` row appears for `get_forecast` with `argumentsSummary = "city=São Paulo, days=30"` and `guardRejectionReason = "days=30 exceeds maximum allowed horizon of 14"`. The row is highlighted orange.
- An `EXECUTED` row immediately follows for `get_forecast` with `argumentsSummary = "city=São Paulo, days=14"` — the agent retried with a corrected argument.
- The job reaches `COMPLETED` with a 14-day forecast.
- The service log shows one guardrail rejection line naming the failed check.

## J4 — Workflow resumes after restart

**Preconditions:** Service running. Mock LLM mode. A job has been submitted and is in `RUNNING` state (the `runAgentStep` is executing tool calls).

**Steps:**
1. Submit a Nairobi query.
2. While the card shows `RUNNING` (after at least one tool call has completed), stop the service (`Ctrl-C` or via Akka MCP `akka_local_stop_service`).
3. Restart the service (`/akka:build` or `akka_local_run_service`).

**Expected:**
- After restart, the job card reappears in `RUNNING` state within 5 s (the workflow journal is durable).
- The workflow resumes from the last completed activity. Tool calls that already completed before the restart do NOT fire again (verify via the service log — the tool fixture methods should not log for the already-completed calls).
- Within 30 s the job transitions to `COMPLETED`.

## J5 — Restricted location blocked

**Preconditions:** Service running. Mock LLM or real LLM. The `RESTRICTED_CITIES` config contains at least one test city (e.g., `TestRestrictedCity`).

**Steps:**
1. In the query textarea, type: "Weather in TestRestrictedCity".
2. Click **Submit query**.

**Expected:**
- The `get_current_weather` tool call is blocked by the guardrail with `guardRejectionReason` mentioning the restricted location.
- If the agent exhausts its 4-iteration budget retrying with the same city, the job transitions to `FAILED` with a clear error record.
- No HTTP call to the weather fixture (or real API) is made for the restricted city — verify via service log.
