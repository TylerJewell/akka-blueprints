# Architecture — lats-tree-search

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SearchEndpoint` is the entry point. It writes a `ProblemSubmitted` event to `ProblemQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `TreeSearchWorkflow` per submission. The workflow alternates two agents — `SearchAgent` expands the current node into candidates, `ReflectorAgent` scores each candidate — and selects the highest-scoring branch. Each step emits events on `SearchTreeEntity`. `TreeView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ProblemSimulator` drips a sample problem every 90 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 45 seconds and records an `EvalRecorded` event for any reflected node that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style solve where the root node expands to three candidates, the reflector scores all three (best score 7, not terminal), the workflow advances the best path to the highest-scoring candidate, then expands that node — and the reflector scores one of its children as terminal (score 9, `isTerminal = true`). The workflow ends in `SOLVED` on the second expansion cycle.

## State machine

The tree moves between two transient states (`EXPANDING`, `REFLECTING`) and two terminal states (`SOLVED`, `BUDGET_EXHAUSTED`). The `REFLECTING → EXPANDING` transition is taken when no candidate in the current batch is terminal and the node count is still below `nodeBudget`. `SOLVED` is reached when any candidate scores `isTerminal = true`. `BUDGET_EXHAUSTED` is reached when the node count hits `nodeBudget` without a terminal node being found or on an irrecoverable agent failure via `defaultStepRecovery`.

## Entity model

`SearchTreeEntity` is the system's source of truth; every transition writes one of eight event types. `ProblemQueue` is the audit log of submissions. `TreeView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 90 s for `expandStep` and `reflectStep` (each individual `REFLECT_NODE` call carries its own 90 s timeout); no timeout on internal selection logic.
- Workflow-wide bound: the workflow bounds itself with `nodeBudget` (default 20). The count increments by `expansionWidth` (default 3) per cycle, so the default budget accommodates 6–7 expansion cycles.
- Default step recovery: `maxRetries(2).failoverTo(exhaustStep)` — any unrecoverable agent failure ends in `BUDGET_EXHAUSTED`, not in a hung workflow.
- Idempotency: `SearchEndpoint.submit` deduplicates on `(taskDescription, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(treeId, nodeId)` so a tick that fires twice for the same node is a no-op.
- Backpropagation: after `selectStep` advances the best path, `BackpropRecorded` events are emitted for non-selected siblings carrying `winnerScore × 0.1` as the delta. This is a view-side annotation; it does not alter the tree's structural state or routing decisions.
- Score threshold: `lats-tree-search.search.score-threshold` (default 8). If all candidates in a reflection batch score below the threshold, the workflow still advances the best path — it does not halt early. A separate halt control would be required for score-gated early stopping.
