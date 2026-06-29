# User journeys — llm-compiler-dag

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path with parallel dispatch

**Preconditions:** Service running on port 9954; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9954/`. App UI tab is visible.
2. In the Query field, type "What is the result of (42 * 17) + the minor version number of the current Akka release?" Click Submit.
3. A new job card appears with status COMPILING.

**Expected:**
- Within 5 s, status transitions to RUNNING via SSE.
- The compilation plan card shows at least three `ToolCall` nodes: one `CALCULATOR` (expression `42 * 17`), one `SEARCH` (Akka release), and one `CALCULATOR` (the addition). The first two nodes have empty `dependsOn`; the third depends on both.
- Both independent nodes dispatch in the same parallel batch. Their `ToolResult` entries have timestamps within 1 second of each other.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows a `QueryAnswer` with a 60–120 word summary that states the final numeric result and cites all three `callId` values.

## J2 — Guardrail blocks an unsafe CODE_EVAL node

**Preconditions:** As J1.

**Steps:**
1. Submit the query "Execute the following Java snippet and return its output: `Runtime.getRuntime().exec('ls')`."

**Expected:**
- The Planner emits a `CompilationPlan` with a `CODE_EVAL` node whose `arguments.code` contains `Runtime`.
- The guardrail's `guardStep` rejects this node; a `ToolCallBlocked` event appears on the job with `callId` of the rejected node.
- The rejected node is marked `SKIPPED` in the DAG view.
- If any other nodes in the plan are independent of the blocked node, they proceed. Otherwise the job transitions to `FAILED` with a clear `failureReason` naming the blocked call. In either case, the `Runtime.getRuntime().exec` call never executes.

## J3 — Operator halt drains the current parallel batch

**Preconditions:** As J1.

**Steps:**
1. Submit a query that produces at least two independent `ToolCall` nodes (e.g., "Search for Akka 3.6.0 release date AND compute 100 * 3.14").
2. While the job status is RUNNING and the first parallel batch is in progress, click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight parallel batch.

**Expected:**
- Both in-flight branches complete normally; their `ToolResult` entries are recorded.
- The next `checkHaltStep` reads `halted=true`, exits the loop, and emits `JobHaltedOperator`.
- Job status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- Jobs submitted after the halt are not dispatched until the operator clicks **Resume**.

## J4 — Secret sanitizer scrubs an exposed key from a LOOKUP result

**Preconditions:** As J1, with the canned fixture `lookup-fixtures.jsonl` containing a key/value entry whose value includes the literal substring `AKIAIOSFODNN7EXAMPLE`.

**Steps:**
1. Submit "Find any stale AWS credentials in the cached configuration lookup and summarise what was found."

**Expected:**
- The Planner emits a `CompilationPlan` with a `LOOKUP` node whose key maps to the fixture containing `AKIAIOSFODNN7EXAMPLE`.
- The simulated LOOKUP tool returns the fixture value including the literal key.
- The `sanitizeStep` replaces `AKIAIOSFODNN7EXAMPLE` with `[REDACTED:aws-access-key]` before the `ToolCallResolved` event is recorded.
- The `ToolResult.output` stored on `JobEntity` does NOT contain `AKIAIOSFODNN7EXAMPLE`.
- The Joiner's synthesis prompt receives only the redacted form.
- The final `QueryAnswer.summary` does not contain the literal key.
- The UI's expanded tool results list renders the redacted span in italics with a tooltip showing `aws-access-key`.
