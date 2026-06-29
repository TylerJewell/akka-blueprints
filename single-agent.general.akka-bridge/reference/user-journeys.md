# User journeys — akka-bridge

## J1 — Submit a search-tools run and get a result

**Preconditions:** Service running on declared port (`http://localhost:9562/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9562/` → App UI tab.
2. From the **Tool set** dropdown, pick `search-tools`.
3. Click **Load seeded example** to fill the task textarea.
4. Click **Submit run**.

**Expected:**
- The new card appears in the live list with status `ACCEPTED` within 1 s.
- The card transitions to `RUNNING` within 1 s. The right-pane tool-call log begins populating.
- Within 30 s the card reaches `COMPLETED`. The tool-call log shows at least one entry with guardrail verdict `PERMITTED` and a non-empty result snippet. The final answer block shows a non-empty `answer` string.
- The budget bar shows the number of permitted calls used out of the declared `toolBudget`.

## J2 — Guardrail blocks a tool not in the permitted set

**Preconditions:** Service running. Any model provider (real or mock).

**Steps:**
1. Submit a run with `Tool set = math-tools` (which defines only `calculate` and `format_number`).
2. In the task textarea, enter a task that would naturally prompt the agent to call `web_search` (e.g., "Search the web for the latest interest rate and then calculate compound interest at that rate for $10,000 over 5 years.").
3. Click **Submit run**.
4. Watch the tool-call log.

**Expected:**
- The agent attempts to call `web_search`.
- The guardrail intercepts the call and records it with verdict `BLOCKED` and rejection reason `tool-not-permitted: web_search is not in the permitted-tools list for this run`.
- `web_search` is never dispatched to `AgentFrameworkAdapter` — no `ToolCallPermitted` event appears for that call.
- The agent receives a tool error, adapts by using only `calculate`, and completes the task with a note that web data was unavailable.
- The final answer block reflects the adaptation.

## J3 — Budget exhaustion transitions the run to BLOCKED

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a run with `Tool set = search-tools` and `Tool budget = 1`.
2. Use the seeded search task (which normally requires 2 tool calls: `web_search` + `summarize`).
3. Click **Submit run**.

**Expected:**
- The first tool call (`web_search`) is permitted and executed; the budget bar shows `1 / 1`.
- The second tool call (`summarize`) is blocked because the budget is exhausted. The guardrail rejection reason reads `budget-exhausted: toolBudget of 1 reached`.
- The entity transitions to `BLOCKED`. The card status pill turns red.
- The final answer block shows the agent's partial answer based on the one permitted call, plus a note that the budget was exhausted before summarisation.

## J4 — Audit log preserves exact arguments (no guardrail rewrite)

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit any run that completes successfully (J1 steps).
2. Wait for `COMPLETED`.
3. Fetch `GET /api/runs/{runId}` and inspect each `toolCalls[].argumentsJson`.
4. Compare the `argumentsJson` values to what the agent sent (visible in the service log at `DEBUG` level with `debug:agent.tool.call`).

**Expected:**
- Each `argumentsJson` in the API response matches exactly what the agent sent — the guardrail did not modify argument payloads for permitted calls.
- `guardrailVerdict` for each call is `PERMITTED`.
- `rejectionReason` is `null` for every permitted call.
- This confirms that `ToolCallGuardrail` acts as an interceptor that records and gates, not as a rewriter.

## J5 — Schema mismatch is caught and reported

**Preconditions:** Service running with mock LLM. The mock includes an entry that sends a tool call with a missing required argument.

**Steps:**
1. Submit a run whose mock entry triggers a `web_search` call without the required `query` argument (i.e., `argumentsJson = "{}"`).
2. Watch the tool-call log.

**Expected:**
- The guardrail catches the missing `query` key against the `web_search` argSchema.
- The call is recorded with verdict `BLOCKED` and rejection reason `schema-mismatch: required key 'query' missing from argumentsJson for tool web_search`.
- The agent receives the tool error, corrects the argument on the next iteration, and the second call (with `query` present) is permitted.
- Both entries appear in the tool-call log in order, showing the failed attempt followed by the corrected permitted attempt.
