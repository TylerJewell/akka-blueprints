# Data model — akka-bridge

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PlannedNode` | `nodeId` | `String` | no | Short stable id unique within the plan. |
| | `nodeType` | `String` | no | Declared node type (e.g., `summarize`, `classify`, `extract`). |
| | `description` | `String` | no | Human-readable description of what this node does. |
| | `estimatedCostTokens` | `int` | no | Estimated token cost for this node's LLM invocation. |
| `ExecutionPlan` | `nodes` | `List<PlannedNode>` | no | Ordered list of nodes to execute. Possibly empty (J6). |
| | `graphId` | `String` | no | The graph id that was submitted. |
| | `plannedAt` | `Instant` | no | When the PLAN task returned. |
| `NodeOutput` | `nodeId` | `String` | no | MUST equal a `PlannedNode.nodeId` from the upstream `ExecutionPlan`. |
| | `rawOutput` | `String` | no | The raw LLM response string before guardrail processing. |
| | `outputSummary` | `String` | no | Paraphrased summary of `rawOutput`. |
| | `guardrailPassed` | `boolean` | no | Set by the runtime after `OutputGuardrail` accepts the response. |
| | `executedAt` | `Instant` | no | When the EXECUTE phase processed this node. |
| `NodeOutputSet` | `outputs` | `List<NodeOutput>` | no | One output per executed node. Possibly empty. |
| | `collectedAt` | `Instant` | no | When the EXECUTE task returned. |
| `NodeStatus` | `nodeId` | `String` | no | Node this status describes. |
| | `status` | `String` | no | `PENDING` / `RUNNING` / `DONE` / `FAILED`. |
| | `message` | `String` | no | Human-readable status message. |
| `ResultItem` | `nodeId` | `String` | no | MUST equal a `NodeOutput.nodeId`. |
| | `heading` | `String` | no | Item heading. |
| | `body` | `String` | no | 2–4 sentences summarising the node's output. |
| | `sourceNodeIds` | `List<String>` | no | Non-empty. Every entry MUST equal a `PlannedNode.nodeId` (E1 rule 3). |
| `FinalResult` | `title` | `String` | no | 1-line title. |
| | `summary` | `String` | no | 1-paragraph summary. |
| | `items` | `List<ResultItem>` | no | `items.size() == nodeOutputs.outputs.size()` (E1 rule 4). |
| | `finalizedAt` | `Instant` | no | When the FINALIZE task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `PolicyChecker` finished. |
| `GuardrailViolation` | `nodeId` | `String` | no | The node whose output was blocked. |
| | `violationType` | `String` | no | `schema-nonconformance` / `prohibited-content` / `hallucinated-reference`. |
| | `reason` | `String` | no | Structured reason from `OutputGuardrail`. |
| | `blockedAt` | `Instant` | no | When the guardrail fired. |
| `GraphRunRecord` (entity state) | `runId` | `String` | no | — |
| | `graphId` | `Optional<String>` | yes | Populated after `GraphRunCreated`. |
| | `plan` | `Optional<ExecutionPlan>` | yes | Populated after `GraphPlanned`. |
| | `nodeOutputs` | `Optional<NodeOutputSet>` | yes | Populated after `NodeExecuted`. |
| | `result` | `Optional<FinalResult>` | yes | Populated after `OutputFinalized`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `GraphRunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `GraphRunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `violations` | `List<GuardrailViolation>` | no | Appended on every `GuardrailViolated` event; empty on the happy path. |

Every nullable lifecycle field on `GraphRunRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`GraphRunStatus`: `CREATED`, `PLANNING`, `PLANNED`, `EXECUTING`, `EXECUTED`, `FINALIZING`, `FINALIZED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `OutputGuardrail`): `PLAN`, `EXECUTE`, `FINALIZE`.

## Events (`GraphRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `GraphRunCreated` | `graphId: String` | → CREATED |
| `PlanStarted` | — | → PLANNING |
| `GraphPlanned` | `plan: ExecutionPlan` | → PLANNED |
| `ExecuteStarted` | — | → EXECUTING |
| `NodeExecuted` | `nodeOutputs: NodeOutputSet` | → EXECUTED |
| `FinalizeStarted` | — | → FINALIZING |
| `OutputFinalized` | `result: FinalResult` | → FINALIZED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailViolated` | `nodeId, violationType, reason, blockedAt` | no status change (audit-only) |
| `GraphRunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `GraphRunRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `violations = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`GraphRunRow` mirrors `GraphRunRecord` exactly. The UI fetches the full row via `GET /api/runs/{id}` and streams updates via `GET /api/runs/sse`.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM graph_run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`GraphTasks.java`)

```java
public final class GraphTasks {
  public static final Task<ExecutionPlan> PLAN_GRAPH = Task
      .name("Plan graph")
      .description("Parse the graph definition and produce an ordered ExecutionPlan with cost estimates")
      .resultConformsTo(ExecutionPlan.class);

  public static final Task<NodeOutputSet> EXECUTE_NODE = Task
      .name("Execute nodes")
      .description("Invoke each planned node in order and collect typed NodeOutputs")
      .resultConformsTo(NodeOutputSet.class);

  public static final Task<FinalResult> FINALIZE_OUTPUT = Task
      .name("Finalize output")
      .description("Aggregate all NodeOutputs into a coherent FinalResult")
      .resultConformsTo(FinalResult.class);

  private GraphTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `PlanTools`, `ExecuteTools`, and `FinalizeTools` carries a `Phase` constant. `OutputGuardrail` is an `after-llm-response` hook — it fires after the LLM returns its response, not before tool calls. The tool registry is built once at startup and is available to `OutputGuardrail` for reference-provenance checks in the FINALIZE phase.
