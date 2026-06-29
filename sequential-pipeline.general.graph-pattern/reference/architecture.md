# Architecture — graph-pattern

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs four tasks in sequence. `GraphEndpoint` accepts a `{description}` POST, writes `RunCreated` onto `GraphRunEntity`, and starts `GraphExecutionWorkflow` keyed by `"graph-" + runId`. The workflow's first step (`parseStep`) emits `ParseStarted`, then calls `GraphAgent` with `TaskDef.taskType(PARSE_REQUEST)` and a `phase = PARSE` metadata tag. The agent invokes `ParseTools.extractIntent` and `ParseTools.identifyConstraints`; every call passes through `DependencyGuardrail` first. Once the agent returns a `ParsedRequest`, the workflow writes `RequestParsed` onto the entity and advances to `planStep` — same pattern, the PLAN task carries `phase = PLAN`. Then `executeStep` runs with `phase = EXECUTE`. Within `executeStep`, the agent processes `GraphNode` entries one by one in topological order, writing a `NodeExecuted{nodeId, output}` event after each successful `runNode` call; `DependencyGuardrail` reads this partial event stream to enforce that each node's declared predecessors are already recorded before the node itself is allowed to run. After `AllNodesExecuted` lands, `mergeStep` runs with `phase = MERGE`. After `OutputsMerged` lands, `evalStep` runs `CoverageScorer` over the recorded `(TaskResult, ExecutionResult, GraphPlan)` triple — no LLM call — and writes `EvaluationScored`. `GraphRunView` projects every event into a read-model row; `GraphEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `CoverageScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. **The task boundary IS the dependency contract across phases.** Between `parseStep` and `planStep`, the workflow writes `RequestParsed` onto the entity. The next step reads `parsedRequest` from the entity to build the PLAN task's instruction context. The agent never sees parse-phase context inside the plan task's conversation; the typed handoff is the only path information travels.

2. **Within EXECUTE, each node is its own dependency checkpoint.** After each `runNode` returns, the workflow writes `NodeExecuted{nodeId, output}` onto the entity before the agent's next iteration. `DependencyGuardrail` reads this growing set of recorded `NodeExecuted` events to decide whether a subsequent `runNode` call is permissible. An out-of-order attempt — calling `runNode(nodeB)` before `NodeExecuted(nodeA)` is recorded, where A is B's declared predecessor — is rejected with a `dependency-violation` error before the tool body runs.

3. **Every tool call on every phase is filtered through `DependencyGuardrail`.** The guardrail's phase check catches misordered phase calls (an EXECUTE tool called during PARSE), and its dependency check catches misordered node executions within EXECUTE. The two checks are independent.

The agent calls themselves are bounded by per-step timeouts (60 s on parse / plan / merge; 120 s on execute to accommodate multi-node DAGs). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → PARSING → PARSED → PLANNING → PLANNED → EXECUTING → EXECUTED → MERGING → MERGED → EVALUATED`.
- Four failure transitions land in `FAILED`: an agent error during `PARSING`, `PLANNING`, `EXECUTING`, or `MERGING`. A `FAILED` run's prior data is preserved on the entity — the UI shows the partial state.
- `DependencyViolated` is a side-event recorded for audit within the `EXECUTING` state; it does not transition status. `NodeExecuted` events are likewise recorded individually within `EXECUTING` as each node completes. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `PUBLISHED` state. The run result is advisory; the user reads it and acts outside the system.

## Entity model

`GraphRunEntity` is the source of truth. It emits thirteen event types — four lifecycle starts, four lifecycle completions (including one per-node `NodeExecuted` plus `AllNodesExecuted`), the evaluation, the dependency-violation audit, the failure, and the initial creation. `GraphRunView` projects every event into a row used by the UI. `GraphExecutionWorkflow` both reads (`getRun`) and writes on the entity across all four phases.

The relationship between `GraphAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload. Within `executeStep`, the incremental `NodeExecuted` writes are the mechanism that keeps `DependencyGuardrail`'s predecessor check accurate in real time.

## Defence-in-depth governance flow

For any run that lands in the entity log, the task description passed through:

1. **Dependency guardrail (phase-level)** — every tool call is filtered by phase. An EXECUTE-phase tool called during PARSE is rejected before the tool body runs; a `DependencyViolated` event records the violation for audit.
2. **Dependency guardrail (node-level, within EXECUTE)** — every `runNode` call is checked against the set of already-recorded `NodeExecuted` events. A node whose predecessors are not yet recorded is rejected; the agent self-corrects by processing nodes in the correct topological order.
3. **GraphAgent (4 task runs)** — four model calls, four structured outputs. Each task's typed result is the dependency handoff to the next phase.
4. **On-decision evaluator** — every emitted `TaskResult` gets a 1–5 coverage-and-traceability score. Node coverage, output traceability, no phantom nodes, and ordering proof are each worth one point on a base of 1.

Each step is independent. The guardrail does not check output quality; the evaluator does not check phase order or node precedence. Removing one of them opens an explicit gap the other does not silently cover.
