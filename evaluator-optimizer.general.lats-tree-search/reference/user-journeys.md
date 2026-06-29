# User journeys — lats-tree-search

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Solve within node budget

**Preconditions:** Service running on port 9913; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9913/`. App UI tab is visible.
2. In the Problem statement field, type "Design a caching strategy for a read-heavy API." Leave the node budget at the default 20. Click Submit.
3. A new tree card appears with status `EXPANDING`.

**Expected:**
- Within a few seconds, the status transitions to `REFLECTING` and the root node's candidate actions appear.
- After the first reflection cycle, the best-path trail shows the highest-scoring candidate selected; sibling nodes show a non-zero backpropagation delta chip.
- The status cycles between `EXPANDING` and `REFLECTING` for each subsequent depth.
- Within the node budget, the tree reaches `SOLVED`: the terminal block shows the terminal node's `actionDescription`, the full best path, and score ≥ 9.
- The expanded view shows every node's expansion, every candidate's score, and the per-node `nodeStatus` pill.

## J2 — Budget exhaustion

**Preconditions:** As J1, plus an override that forces the Reflector to never return `isTerminal = true` (test mode — submit the literal problem `"test-force-exhaust"`, which the mock provider's `seedFor` logic always answers with `isTerminal = false` and score ≤ 6).

**Steps:**
1. Submit the problem `"test-force-exhaust"` with node budget 9 (3 expansion cycles of width 3).

**Expected:**
- Tree progresses through 3 expansion + 3 reflection cycles (9 nodes total).
- After the 9th node is reflected without a terminal hit, the tree transitions to `BUDGET_EXHAUSTED`.
- The terminal block shows the highest-scoring partial node's `actionDescription`, the best partial path, and `exhaustionReason = "node budget reached (9)"`.
- All 9 nodes are present in the expanded view, each with its `actionDescription`, score, and `nodeStatus`.
- `GET /api/trees/{id}` returns the full `SearchTree` with `status: "BUDGET_EXHAUSTED"` and all 9 nodes in `nodes[]`.

## J3 — Backpropagation visible in the UI

**Preconditions:** At least one tree has completed at least one reflection cycle (any status beyond initial `EXPANDING`).

**Steps:**
1. Click a tree card to expand it.
2. Locate a node with `nodeStatus = PRUNED` (a non-selected candidate from any reflection cycle).

**Expected:**
- The pruned node's card shows a `backpropDelta` chip with a non-zero value (e.g., "Δ 0.7").
- The selected sibling (the one that was advanced to) shows `nodeStatus = SELECTED` and the score that produced the delta.
- The backprop deltas match the formula `winnerScore × 0.1` for each sibling.
- The best-path trail reflects only `SELECTED` and `TERMINAL` nodes.

## J4 — Eval-event timeline

**Preconditions:** At least one tree has completed at least one reflection cycle.

**Steps:**
1. Wait up to 45 s after any node is reflected (EvalSampler tick interval).
2. Click the tree card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per reflected node, with `nodeId`, `depth`, `score`, `isTerminal`, and `backpropDelta` populated.
- The terminal transition (`TreeSolved` or `BudgetExhausted`) also appears as a final `EvalRecorded` event carrying the total node count and best partial or terminal score.
- `GET /api/trees/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
