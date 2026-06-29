# Data model — graph-pattern

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ParsedRequest` | `intent` | `String` | no | The extracted goal from the task description. |
| | `constraints` | `List<String>` | no | Possibly empty; J5 demonstrates the empty-constraints path. |
| | `parsedAt` | `Instant` | no | When the PARSE task returned. |
| `GraphNode` | `nodeId` | `String` | no | Slug. MUST be unique within a `GraphPlan`. |
| | `label` | `String` | no | Human-readable node name. |
| | `predecessorIds` | `List<String>` | no | Empty for root nodes. Each entry MUST equal a `nodeId` in the same `GraphPlan`. |
| `Edge` | `fromNodeId` | `String` | no | MUST equal a `nodeId` in `GraphPlan.nodes`. |
| | `toNodeId` | `String` | no | MUST equal a `nodeId` in `GraphPlan.nodes`. |
| `GraphPlan` | `nodes` | `List<GraphNode>` | no | Possibly empty; J5 demonstrates the 1-node case. |
| | `edges` | `List<Edge>` | no | Possibly empty (a single-node plan has no edges). |
| | `plannedAt` | `Instant` | no | When the PLAN task returned. |
| `NodeInput` | `nodeId` | `String` | no | The node being executed. |
| | `predecessorOutputIds` | `List<String>` | no | Empty for root nodes. Each entry MUST equal an `outputId` of a predecessor's `NodeOutput`. |
| `NodeOutput` | `nodeId` | `String` | no | MUST equal a `nodeId` in the `GraphPlan`. |
| | `outputId` | `String` | no | Short stable id (`out-<8 hex>`). Unique within a run. |
| | `body` | `String` | no | The node's produced content, 1–4 sentences. |
| | `sourceNodeIds` | `List<String>` | no | MUST equal the node's `predecessorIds` from the `GraphPlan`. |
| `ExecutionResult` | `outputs` | `List<NodeOutput>` | no | One entry per `GraphNode`. |
| | `executionOrder` | `List<String>` | no | Node IDs in execution order. Must be topologically consistent with declared edges. |
| | `executedAt` | `Instant` | no | When the EXECUTE task returned. |
| `OutputRef` | `nodeId` | `String` | no | MUST equal a `nodeId` in `GraphPlan.nodes` (no phantom nodes — E1 rule 3). |
| | `outputId` | `String` | no | MUST equal a `NodeOutput.outputId` in `ExecutionResult.outputs` (E1 rule 2). |
| `TaskResult` | `title` | `String` | no | 1-line title. |
| | `summary` | `String` | no | 1-paragraph summary. |
| | `outputRefs` | `List<OutputRef>` | no | `outputRefs.length == plan.nodes.length` (E1 rule 1). |
| | `mergedAt` | `Instant` | no | When the MERGE task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `CoverageScorer` finished. |
| `DependencyViolation` | `phase` | `String` | no | `PARSE` / `PLAN` / `EXECUTE` / `MERGE` for phase-level violations; `EXECUTE` for node-level violations. |
| | `nodeId` | `String` | no | Target node of the rejected call (phase-level violations carry the tool name here). |
| | `missingPredecessors` | `List<String>` | no | Node IDs not yet recorded at the time of rejection. Empty for phase-level violations. |
| | `reason` | `String` | no | Structured reason from `DependencyGuardrail`. |
| | `violatedAt` | `Instant` | no | When the guardrail fired. |
| `RunRecord` (entity state) | `runId` | `String` | no | — |
| | `description` | `Optional<String>` | yes | Populated after `RunCreated`. |
| | `parsedRequest` | `Optional<ParsedRequest>` | yes | Populated after `RequestParsed`. |
| | `plan` | `Optional<GraphPlan>` | yes | Populated after `GraphPlanned`. |
| | `executionResult` | `Optional<ExecutionResult>` | yes | Populated after `AllNodesExecuted`. |
| | `taskResult` | `Optional<TaskResult>` | yes | Populated after `OutputsMerged`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `RunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `dependencyViolations` | `List<DependencyViolation>` | no | Appended on every `DependencyViolated` event; empty on the happy path. |
| | `nodeExecutedMap` | `Map<String, NodeOutput>` | no | Populated incrementally as `NodeExecuted` events arrive; used by `DependencyGuardrail` for predecessor checks. |

Every nullable lifecycle field on `RunRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RunStatus`: `CREATED`, `PARSING`, `PARSED`, `PLANNING`, `PLANNED`, `EXECUTING`, `EXECUTED`, `MERGING`, `MERGED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `DependencyGuardrail`): `PARSE`, `PLAN`, `EXECUTE`, `MERGE`.

## Events (`GraphRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunCreated` | `description: String` | → CREATED |
| `ParseStarted` | — | → PARSING |
| `RequestParsed` | `parsedRequest: ParsedRequest` | → PARSED |
| `PlanStarted` | — | → PLANNING |
| `GraphPlanned` | `plan: GraphPlan` | → PLANNED |
| `ExecuteStarted` | — | → EXECUTING |
| `NodeExecuted` | `nodeId: String, output: NodeOutput` | no status change (within EXECUTING — incremental) |
| `AllNodesExecuted` | `executionResult: ExecutionResult` | → EXECUTED |
| `MergeStarted` | — | → MERGING |
| `OutputsMerged` | `taskResult: TaskResult` | → MERGED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `DependencyViolated` | `phase, nodeId, missingPredecessors, reason, violatedAt` | no status change (audit-only) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `RunRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `dependencyViolations = List.of()`, `nodeExecutedMap = Map.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RunRow` mirrors `RunRecord` exactly. The UI fetches the full row via `GET /api/runs/{id}` and streams updates via `GET /api/runs/sse`.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM graph_run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`GraphTasks.java`)

```java
public final class GraphTasks {
  public static final Task<ParsedRequest> PARSE_REQUEST = Task
      .name("Parse request")
      .description("Extract intent and constraints from the task description")
      .resultConformsTo(ParsedRequest.class);

  public static final Task<GraphPlan> PLAN_GRAPH = Task
      .name("Plan graph")
      .description("Build a DAG of processing nodes with explicit predecessor edges")
      .resultConformsTo(GraphPlan.class);

  public static final Task<ExecutionResult> EXECUTE_NODES = Task
      .name("Execute nodes")
      .description("Run each node in topological order, gated on predecessor outputs")
      .resultConformsTo(ExecutionResult.class);

  public static final Task<TaskResult> MERGE_OUTPUTS = Task
      .name("Merge outputs")
      .description("Aggregate node outputs into a unified TaskResult")
      .resultConformsTo(TaskResult.class);

  private GraphTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ParseTools`, `PlanTools`, `ExecuteTools`, and `MergeTools` carries a `Phase` constant. `DependencyGuardrail` reads this constant for the phase check and, within `EXECUTE` phase, reads the target `nodeId` from the tool arguments and verifies all predecessor nodes have `NodeExecuted` events recorded on the entity. The tool registry is built once at startup; the guardrail reads it for every call.

## Node-dependency invariant

`NodeExecuted` events are written to `GraphRunEntity` individually inside `executeStep`, one per node, as each `runNode` tool call returns. `AllNodesExecuted` is written once all `NodeExecuted` events are committed. `DependencyGuardrail` reads the entity's `nodeExecutedMap` (populated by the `NodeExecuted` applier) to answer the question: "have all declared predecessors of node X been executed?" This is the runtime mechanism that makes the graph primitive observable and enforceable at the entity boundary.
