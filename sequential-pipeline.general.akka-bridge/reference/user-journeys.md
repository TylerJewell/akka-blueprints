# User journeys — akka-bridge

## J1 — Submit a graph and get a final result

**Preconditions:** Service running on declared port (`http://localhost:9994/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded graph `summarize-and-classify` has a matching `src/main/resources/sample-data/graphs/summarize-and-classify.json` file.

**Steps:**
1. Open `http://localhost:9994/` → App UI tab.
2. From the **Pick a seeded graph** dropdown, pick `summarize-and-classify`.
3. Click **Run graph**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PLANNING` within 1 s more.
- Within ~20 s the card reaches `PLANNED`. The right pane shows the execution plan table with ≥ 2 rows; each row has a non-empty nodeId, nodeType, and description.
- Within ~20 s more the card reaches `EXECUTED`. The right pane shows ≥ 2 node outputs; every output has a `guardrailPassed: true` badge and a non-empty outputSummary.
- Within ~20 s more the card reaches `FINALIZED`, then `EVALUATED` within 1 s of that. The right pane shows a `FinalResult` with `items.length == plan.nodes.length`, every item has at least one sourceNodeId from the plan, and the eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — After-LLM-response guardrail blocks a policy-violating output

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `execute-nodes.json` includes one entry whose first `NodeOutput.rawOutput` contains a prohibited-content pattern — this is the deliberately policy-violating entry.

**Steps:**
1. Submit any seeded graph three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/runs/sse`).

**Expected:**
- On the third run's `executeStep`, the agent's first LLM response contains a prohibited pattern. `OutputGuardrail` blocks it; a `GuardrailViolated{nodeId: "summarize-001", violationType: "prohibited-content", reason: "content-violation: ..."}` event lands on the entity.
- The blocked response is NEVER written to the entity or forwarded — there is no log line from `GraphRunEntity.recordNodeOutputs` for that iteration.
- The agent's second iteration produces a clean response. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a violation fired. The violation-log strip on the right pane shows the one blocked output with its full structured reason.

## J3 — Hallucinated node reference flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `finalize-output.json` includes one entry whose first `ResultItem.sourceNodeIds` contains a nodeId absent from the paired `ExecutionPlan`.

**Steps:**
1. Submit any seeded graph six times. (The hallucinated-reference entry is selected once in every six runs by the mock's `seedFor(runId)` modulo.)
2. Watch the live list until a run's card border highlights red.

**Expected:**
- The flagged run lands well-formed (the guardrail checks content policy, not reference provenance at execution time — provenance is caught by the evaluator).
- The eval score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Reference provenance failed: item 'X' references nodeId 'Y' which does not appear in the ExecutionPlan."*
- The card's border highlights red. The reviewer knows to inspect this run before acting on its output.
- The other five runs in the batch scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded graph.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the runId.

**Expected:**
- The PLAN task's log entries show only `parseGraphDefinition` and `estimateNodeCost` calls.
- The EXECUTE task's log entries show only `invokeNode` and `checkNodeStatus` calls.
- The FINALIZE task's log entries show only `aggregateOutputs` and `formatResult` calls.
- No cross-phase tool calls appear.
- The order of tasks in the log is PLAN → EXECUTE → FINALIZE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Output parity: item count equals node count

**Preconditions:** Service running. Any model provider. The seeded graph `extract-and-enrich` has 4 planned nodes in its fixture file.

**Steps:**
1. Submit the seeded graph `extract-and-enrich`.
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `FinalResult.items.length` equals the recorded `ExecutionPlan.nodes.length` (both 4).
- Every `item.nodeId` matches one `nodeOutput.nodeId` from the `NodeOutputSet`, one-to-one.
- The presence of an eval score of 5 with rationale "all checks passed" confirms output parity held.

## J6 — Unknown graph id with no matching fixture

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/graphs/<custom-id>.json` exists for the user's graph id.

**Steps:**
1. In the App UI, type a custom graph id (e.g. `my-custom-pipeline`) into the graph id field.
2. Click **Run graph**.

**Expected:**
- `PlanTools.parseGraphDefinition` returns an empty list (no matching fixture).
- The agent's PLAN task returns an `ExecutionPlan` with `nodes = []`.
- The workflow advances to `executeStep`. The EXECUTE task returns a `NodeOutputSet` with `outputs = []`.
- The workflow advances to `finalizeStep`. The FINALIZE task returns a `FinalResult` with `title = "(no node outputs)"` and `items = []`.
- The eval score chip shows 1 (no nodes, no outputs, no items — parity is technically satisfied but no other rules score). The rationale names "no nodes to score."
- The pipeline completes; nothing crashes; the empty result is honestly empty.
