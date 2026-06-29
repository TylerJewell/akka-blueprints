# User journeys — graph-pattern

## J1 — Submit a task and get a result

**Preconditions:** Service running on declared port (`http://localhost:9145/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded task `Summarize recent Akka release notes` has a matching `src/main/resources/sample-data/tasks/summarize-akka-release-notes.json` file.

**Steps:**
1. Open `http://localhost:9145/` → App UI tab.
2. From the **Pick a seeded task** dropdown, pick `Summarize recent Akka release notes`.
3. Click **Run graph**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PARSING` within 1 s more.
- Within ~20 s the card reaches `PLANNED`. The right pane shows the Graph plan table with ≥ 2 nodes and the matching edge list. Every node has a non-empty label; every edge references valid node IDs.
- Within ~30 s the card reaches `EXECUTED`. The right pane's Execution panel shows per-node status chips, all `DONE`. Each node's body is non-empty and every non-root node's `sourceNodeIds` matches its declared `predecessorIds`.
- Within ~10 s more the card reaches `EVALUATED`. The right pane shows the `TaskResult` with `outputRefs.length == plan.nodes.length` and an eval score chip showing 5/5.
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Dependency guardrail blocks an out-of-order node execution

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `execute-nodes.json` includes one entry whose `tool_calls` array attempts `runNode` on a non-root node before its predecessor's `runNode` — this is the deliberately out-of-order entry.

**Steps:**
1. Submit any seeded task three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/runs/sse`).

**Expected:**
- On the third submission's `executeStep`, the agent's first iteration calls `runNode(summarize)` before `NodeExecuted(fetch-releases)` has been recorded on the entity. `DependencyGuardrail` rejects it; a `DependencyViolated{phase: "EXECUTE", nodeId: "summarize", missingPredecessors: ["fetch-releases"], reason: "dependency-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `ExecuteTools.runNode` — there is no log line from the tool body for that invocation.
- The agent's second iteration processes nodes in correct topological order (`fetch-releases` first, then `summarize`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a violation fired. The dependency-violation log strip on the right pane shows the one rejected call with its full structured reason and missing predecessor list.

## J3 — Phantom node reference flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `merge-outputs.json` includes one entry whose `TaskResult.outputRefs` list references a `nodeId` absent from the paired `GraphPlan.nodes`.

**Steps:**
1. Submit any seeded task six times. (The phantom-node entry is selected once in every six runs by the mock's `seedFor(runId)` modulo.)
2. Watch the live list until a run's card border highlights red.

**Expected:**
- The flagged run lands well-formed (the dependency guardrail only checks phase order and node predecessors, not output references in the merge result).
- The eval score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: `"No phantom nodes failed: outputRefs[0].nodeId 'phantom-node' does not appear in GraphPlan.nodes."`.
- The card's border highlights red. The reader knows to inspect this result before acting.
- The other five runs in the batch scored ≥ 4.

## J4 — Execution order matches declared edges

**Preconditions:** Service running. Any model provider. The seeded task `Classify incoming support tickets by severity` has a 3-node DAG with a fan-out edge (one root node feeding two parallel leaf nodes, though the agent processes them sequentially as a topo-sorted list).

**Steps:**
1. Submit `Classify incoming support tickets by severity`.
2. Wait for `EVALUATED`.
3. Read the `executionResult.executionOrder` field from `GET /api/runs/{id}`.

**Expected:**
- `executionOrder` has exactly 3 entries matching the 3 planned nodes.
- For every edge `(A → B)` in `plan.edges`, A appears before B in `executionOrder`.
- The eval score chip shows 5/5 with rationale `"Node coverage, output traceability, no phantom nodes, and ordering proof all satisfied."`.
- If the agent had produced an ordering that violated any declared edge, the `CoverageScorer` ordering-proof check would have set the score to 4 with a rationale naming the violating pair. The 5/5 score is the empirical proof that the ordering held.

## J5 — Custom task description with no matching sample file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/tasks/<slug>.json` exists for the user's description.

**Steps:**
1. In the App UI, type a custom description (e.g. `Analyse the nutritional value of medieval peasant bread`) into the task field.
2. Click **Run graph**.

**Expected:**
- `ParseTools.extractIntent` returns a best-effort intent string derived from the description text; `identifyConstraints` returns an empty list.
- The PLAN task builds a minimal 1-node DAG (a single root node with the extracted intent as its label and no predecessors).
- The EXECUTE task runs the single root node; `ExecutionResult.executionOrder` has one entry.
- The MERGE task produces a `TaskResult` with one `OutputRef` matching the single node.
- The eval score chip shows 5/5 — all four rules are satisfied for a 1-node DAG. The run completes without crashing.

## J6 — Phase-gate guardrail blocks a cross-phase tool call during PARSE

**Preconditions:** Mock LLM mode. The mock's `parse-request.json` includes one entry whose `tool_calls` array starts with `runNode(...)` — an EXECUTE-phase tool called during PARSE.

**Steps:**
1. Submit any seeded task three times. (The violating entry fires on every 3rd run per mock's `seedFor(runId)` modulo.)
2. Inspect the `GET /api/runs/sse` stream for the 3rd run.

**Expected:**
- On the 3rd run's `parseStep`, the agent's first iteration calls `runNode(...)`. `DependencyGuardrail` rejects it with `phase-violation: runNode requires phase EXECUTE, saw PARSING`.
- A `DependencyViolated` event lands on the entity. The tool body never executes.
- The agent's second iteration falls through to a normal parse sequence (`extractIntent` + `identifyConstraints`). The run completes as in J1.
- The dependency-violation log strip on the run detail shows the one rejected call, distinguishing it from a within-EXECUTE ordering violation by its `phase` field (`PARSE` vs `EXECUTE`).
