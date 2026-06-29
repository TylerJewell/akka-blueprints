# User journeys — akka-stategraph-bridge

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: linear graph with conditional edge

**Preconditions:** Service running on port 9454; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9454/`. App UI tab is visible.
2. In the graph definition textarea, paste the 4-node graph from `sample-events/graph-definitions.jsonl` (the second entry: nodes `fetch-docs → analyse → decide → output`, with a conditional edge from `decide` to either `retry` or `output` based on `error_code == null`).
3. Click Submit.
4. A new run card appears with status `PLANNING`.

**Expected:**
- Within 5 s, status transitions to `EXECUTING` via SSE.
- Within ~3 minutes, status transitions to `COMPLETED`.
- The expanded view shows:
  - A graph plan with 4 nodes, 3 edges (including the conditional edge). Entry node `fetch-docs`; terminal node `output`.
  - A current state with at least `release_version` and `summary_text` populated.
  - An execution trace with one entry per node visited (3–5 entries). Every entry has a non-empty `scrubbedContent`.
  - A `RunResult` with a non-empty `summary` and `traceLength` matching the trace.

## J2 — Guardrail blocks a non-allow-listed tool annotation

**Preconditions:** As J1.

**Steps:**
1. Submit the fifth sample graph definition, which contains a node whose `toolAnnotation` names `external-api.example.com/write`.

**Expected:**
- The workflow's `guardrailStep` rejects the node; a `NodeBlocked` trace entry appears with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker` citing the non-allow-listed host.
- The workflow calls `PlannerAgent.REVISE_TOOL_BINDING`. If the planner returns a valid revised binding, execution continues from that node with the revised tool.
- If the planner cannot find a valid revision, the run ends in `FAILED` with a clear `failureReason` of `"tool binding revision failed: <nodeId>"`. Either outcome is acceptable; the offending tool call must never execute.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit the 5-node parallel-branch graph (fourth sample definition).
2. While the run status is `EXECUTING` (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight node execution complete.

**Expected:**
- The in-flight `NodeExecuted` trace entry is recorded normally.
- The next `checkHaltStep` reads `halted=true`, exits the loop, and emits `RunHaltedOperator`.
- Run status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — Secret sanitizer scrubs an exposed key from node output

**Preconditions:** As J1, with the canned fixture `sample-data/node-fixtures.jsonl` containing one entry whose `rawContent` includes `AKIAIOSFODNN7EXAMPLE`.

**Steps:**
1. Submit the sixth sample graph definition, whose second node's fixture lookup surfaces the legacy config fixture.

**Expected:**
- The `NodeAgent` returns `rawContent` containing `AKIAIOSFODNN7EXAMPLE`.
- The `sanitizeStep` replaces it with `[REDACTED:aws-access-key]` before `recordStateStep` runs.
- The `TraceEntry.scrubbedContent` for that step does NOT contain `AKIAIOSFODNN7EXAMPLE`.
- `GraphState` fields merged from that node's `updatedFields` do not contain the literal key.
- The `EdgeRouterAgent`'s next routing prompt sees only the redacted form.
- The UI's expanded trace renders the redacted span in italics with a tooltip showing `aws-access-key`.
