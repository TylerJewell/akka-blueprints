# Data model — lats-tree-search

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Problem` | `taskDescription` | `String` | no | The user-submitted problem statement. |
| | `nodeBudget` | `int` | no | Maximum number of tree nodes to expand before halting. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `NodeExpansion` | `nodeId` | `String` | no | The node being expanded. |
| | `candidates` | `List<CandidateNode>` | no | Exactly `expansionWidth` candidates. |
| | `expandedAt` | `Instant` | no | When the SearchAgent returned. |
| `CandidateNode` | `candidateId` | `String` | no | Short slug for this candidate (e.g., `root-a`). |
| | `actionDescription` | `String` | no | One-sentence description of the next action. |
| | `reasoning` | `String` | no | Two-to-four sentence justification. |
| `ReflectionNote` | `rationale` | `String` | no | One sentence explaining the score. |
| | `actionability` | `String` | no | One sentence describing the next expansion step if not terminal. |
| `NodeScore` | `candidateId` | `String` | no | The candidate this score belongs to. |
| | `score` | `int` | no | 1–10 rubric score. |
| | `isTerminal` | `boolean` | no | `true` when score ≥ 9 and candidate is a complete solution. |
| | `note` | `ReflectionNote` | no | Rationale and actionability. |
| | `backpropDelta` | `double` | no | `score / 10.0`; distributed to sibling nodes. |
| | `reflectedAt` | `Instant` | no | When the ReflectorAgent returned. |
| `TreeNode` | `nodeId` | `String` | no | Unique node identifier. |
| | `parentNodeId` | `String` | no | Parent node id; empty string at root. |
| | `depth` | `int` | no | 0 at root. |
| | `actionDescription` | `String` | no | The action this node represents. |
| | `score` | `Optional<NodeScore>` | yes | Empty until reflected. |
| | `nodeStatus` | `NodeStatus` | no | See enum. |
| `SearchTree` (entity state) | `treeId` | `String` | no | Unique id. |
| | `taskDescription` | `String` | no | Original problem statement. |
| | `nodeBudget` | `int` | no | Maximum node count (default 20). |
| | `expansionWidth` | `int` | no | Candidates per expansion (default 3). |
| | `status` | `TreeStatus` | no | See enum. |
| | `nodes` | `List<TreeNode>` | no | Bounded at `nodeBudget`; starts with root node. |
| | `bestPath` | `List<String>` | no | Ordered list of selected `nodeId`s. |
| | `terminalNodeId` | `Optional<String>` | yes | Populated on `TreeSolved`. |
| | `terminalContent` | `Optional<String>` | yes | Populated on `TreeSolved`. |
| | `exhaustionReason` | `Optional<String>` | yes | Populated on `BudgetExhausted`. |
| | `createdAt` | `Instant` | no | When `TreeCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the tree reached a terminal state. |

## Enums

`TreeStatus`: `EXPANDING`, `REFLECTING`, `SOLVED`, `BUDGET_EXHAUSTED`.

`NodeStatus`: `PENDING`, `SELECTED`, `PRUNED`, `TERMINAL`.

## Events (`SearchTreeEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TreeCreated` | `taskDescription, nodeBudget, expansionWidth, createdAt` | Workflow `startStep` | → `EXPANDING` |
| `NodeExpanded` | `nodeId, candidates: List<CandidateNode>, expandedAt` | After `expandStep` returns | (no status change; adds candidate nodes as `PENDING`) |
| `NodeReflected` | `candidateId, score: NodeScore` | After each `REFLECT_NODE` call | → `REFLECTING` (on first reflection in a batch); populates `nodes[candidateId].score` |
| `BestPathAdvanced` | `selectedNodeId, depth` | After `selectStep` picks the highest-scoring candidate | (no status change; appends to `bestPath`, sets `selectedNodeId.nodeStatus = SELECTED`) |
| `BackpropRecorded` | `siblingNodeId, delta: double` | After `selectStep` for each non-selected sibling | (no status change; annotates sibling node with `backpropDelta`) |
| `TreeSolved` | `terminalNodeId, terminalContent, bestPath` | `NodeScore.isTerminal = true` | → `SOLVED`, `finishedAt = now` |
| `BudgetExhausted` | `bestPartialNodeId, bestPartialContent, exhaustionReason` | `nodes.size() >= nodeBudget` AND no terminal; OR `defaultStepRecovery` failover | → `BUDGET_EXHAUSTED`, `finishedAt = now` |
| `EvalRecorded` | `nodeId, depth, score, isTerminal, backpropDelta, recordedAt` | `EvalSampler` per reflected node; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`ProblemQueue`)

| Event | Payload |
|---|---|
| `ProblemSubmitted` | `treeId, taskDescription, nodeBudget, submittedBy, submittedAt` |

## View row

`TreeRow` is structurally identical to `SearchTree` — the `nodes` list is bounded at `nodeBudget` (default 20) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SearchTreeEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllTrees` returning the full list. Callers filter by `status` client-side.
