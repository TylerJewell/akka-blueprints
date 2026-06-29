# User journeys — react-loop-agent

## J1 — Submit an arithmetic query and follow the step trace

**Preconditions:** Service running on `http://localhost:9702/`; a valid model-provider API key set, or the mock LLM selected at scaffold time. Calculator tool enabled; no tools denied.

**Steps:**
1. Open `http://localhost:9702/` → App UI tab.
2. From the **Seeded query** dropdown, pick `Arithmetic (3-step)`.
3. Confirm Calculator is checked in the tool checklist.
4. Click **Run**.

**Expected:**
- The new run card appears in the live list with status `PENDING` within 1 s.
- The card transitions to `RUNNING` within 1 s. The step trace in the right pane begins filling in: a THOUGHT chip, then an ACTION chip referencing `Calculator`, then a green OBSERVATION chip with the tool output.
- The trace continues with at least two Action/Observation pairs.
- Within 30 s the card reaches `COMPLETED`. The final ANSWER step appears in the trace, and the right pane shows the final answer text block.
- Within 1 s of `COMPLETED`, the chain score chip appears (1–5) plus a one-line rationale.

## J2 — Denied tool triggers guardrail block and agent recovery

**Preconditions:** Service running. `DataFetch` is in the deny list for the submitted run (entered in the denied tools field or pre-configured in the seeded query).

**Steps:**
1. Pick the `Conditional lookup` seeded query, which is designed to attempt `DataFetch`.
2. In the denied tools field, confirm `DataFetch` is listed.
3. Click **Run**.
4. Watch the step trace in the right pane via the live SSE stream.

**Expected:**
- The agent emits a THOUGHT then an ACTION step calling `DataFetch`.
- The ACTION step card shows a red `[BLOCKED]` chip; the `ActionBlocked` event is recorded in the entity.
- The following step is an OBSERVATION card reading `[BLOCKED] Tool DataFetch is on the deny list for this run.`
- The agent emits another THOUGHT acknowledging the block, then selects a permitted tool (or returns a partial ANSWER if no alternative applies).
- The run reaches `COMPLETED` or `EXHAUSTED` — it does not reach `FAILED`. The UI never displays a raw exception; the block is surfaced as a structured step in the trace.

## J3 — Circular reasoning chain scores low on eval

**Preconditions:** Mock LLM mode. The mock entry for the relevant run seed emits the same THOUGHT text twice without an ACTION between them.

**Steps:**
1. Submit a query using the mock that is configured to produce a circular THOUGHT pattern (see `src/main/resources/mock-responses/run-query.json` "circular-loop" entry).
2. Wait for `COMPLETED` or `EXHAUSTED`.

**Expected:**
- The chain score chip shows **1** or **2** and the rationale reads text similar to: "Circular Thought detected at step N: identical to step M."
- The run card's border highlights red in the live list.
- The full step trace is still visible; the duplicated THOUGHT steps are both present, confirming the scorer caught a real pattern rather than a false positive.

## J4 — Iteration budget exhaustion transitions to EXHAUSTED

**Preconditions:** Mock LLM mode configured to emit 10 THOUGHT/ACTION/OBSERVATION cycles with no ANSWER step.

**Steps:**
1. Submit a query using the mock "no-answer" entry.
2. Watch the step trace fill to 10 steps.

**Expected:**
- After the 10th step is recorded, the run card transitions to `EXHAUSTED` (not `FAILED`).
- The right pane shows the full 10-step trace with no ANSWER chip.
- The outcome badge reads `EXHAUSTED`.
- Within 1 s the chain score chip appears; the rationale notes the budget was reached.
- The UI status pill is orange (EXHAUSTED), not red (FAILED).

## J5 — Unrecoverable workflow error transitions to FAILED

**Preconditions:** Service running. Simulate a workflow error by submitting a malformed tool manifest (e.g., a `paramSchema` that is not valid JSON) that causes the loopStep to throw after retry exhaustion.

**Steps:**
1. POST directly to `/api/runs` with a `paramSchema` field containing `"{bad json"`.
2. Wait for the run to process.

**Expected:**
- The workflow's loopStep attempts the call, fails validation inside `ToolCallGuardrail`, retries per `maxRetries(2)`, then fails over to the error step.
- The run card transitions to `FAILED`.
- The UI status pill is red.
- The partial trace is preserved; any steps recorded before the error are visible.
- No stack trace appears in the UI; the error step writes a plain-text `reason` string to `RunEntity.fail`.

## J6 — Seeded research query uses multiple tools in sequence

**Preconditions:** Service running. Calculator, WebLookup, and DataFetch all enabled; no denials.

**Steps:**
1. Pick the `Research synthesis` seeded query (requires WebLookup then DataFetch).
2. Click **Run**.
3. Wait for `COMPLETED`.

**Expected:**
- The step trace shows at least one WebLookup ACTION/OBSERVATION pair followed by at least one DataFetch ACTION/OBSERVATION pair.
- The final ANSWER step references both tool names.
- The chain score is 4 or 5; the rationale confirms both tool claims are supported by observations.
- Each OBSERVATION chip in the trace shows the canned stub response for the keyword/record id used.
