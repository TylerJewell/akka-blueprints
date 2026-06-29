# Data model — akka-stategraph-bridge

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GraphDefinitionRequest` | `graphJson` | `String` | no | Raw graph definition submitted by the user. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `NodeDef` | `nodeId` | `String` | no | Unique identifier within the graph. |
| | `description` | `String` | no | Human-readable description of the node's purpose. |
| | `toolAnnotation` | `Optional<String>` | yes | `<host>/<function>` form; absent if the node uses no external tool. |
| | `maxVisits` | `int` | no | Maximum times this node may be visited in one run (default 3). |
| `EdgeDef` | `fromNodeId` | `String` | no | Source node. |
| | `toNodeId` | `String` | no | Target node. |
| | `predicate` | `Optional<String>` | yes | Boolean expression over `GraphState.fields`; absent = unconditional. |
| | `isBackEdge` | `boolean` | no | True if this edge creates a cycle. |
| `GraphPlan` | `nodes` | `List<NodeDef>` | no | All nodes normalised from the submitted definition. |
| | `edges` | `List<EdgeDef>` | no | All edges normalised from the submitted definition. |
| | `entryNodeId` | `String` | no | First node to execute. |
| | `terminalNodeIds` | `List<String>` | no | Nodes that end the run on arrival. |
| | `cycleAnnotations` | `List<String>` | no | Human-readable notes on back-edges and cycle budgets. |
| `NodeDispatch` | `nodeId` | `String` | no | Node being dispatched. |
| | `toolAnnotation` | `Optional<String>` | yes | Guardrail-verified annotation; absent for non-tool nodes. |
| | `currentState` | `GraphState` | no | State snapshot passed to `NodeAgent`. |
| `NodeOutput` | `nodeId` | `String` | no | Echo of the dispatched node. |
| | `updatedFields` | `Map<String, Object>` | no | Fields the node adds to or overwrites in `GraphState`. |
| | `rawContent` | `String` | no | Free-text summary of what the node did (pre-sanitize). |
| | `ok` | `boolean` | no | True if the node fulfilled its description. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `RoutingDecision` | `fromNodeId` | `String` | no | Node whose outgoing edges were evaluated. |
| | `nextNodeId` | `Optional<String>` | yes | Next node; absent when `terminal=true` or routing failed. |
| | `terminal` | `boolean` | no | True when the selected target is a terminal node. |
| | `rationale` | `String` | no | One sentence explaining the routing choice. |
| `TraceEntry` | `stepIndex` | `int` | no | 1-based sequential index across the entire run. |
| | `nodeId` | `String` | no | Node that was dispatched (or blocked). |
| | `verdict` | `NodeExecutionVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / SANITIZED. |
| | `scrubbedContent` | `String` | no | Sanitized content of the node output. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED_BY_GUARDRAIL or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `GraphState` | `fields` | `Map<String, Object>` | no | Accumulates updated fields from each executed node. |
| `RunResult` | `summary` | `String` | no | 60–120 word summary of the run outcome. |
| | `finalState` | `GraphState` | no | State snapshot at run completion. |
| | `traceLength` | `int` | no | Total number of trace entries. |
| | `producedAt` | `Instant` | no | When the result was produced. |
| `GraphRun` (entity state) | `runId` | `String` | no | Unique run id. |
| | `graphJson` | `String` | no | Original graph definition JSON. |
| | `status` | `RunStatus` | no | See enum. |
| | `plan` | `Optional<GraphPlan>` | yes | Populated after `RunPlanned`. |
| | `currentState` | `Optional<GraphState>` | yes | Populated after first `NodeExecuted`. |
| | `trace` | `Optional<List<TraceEntry>>` | yes | Populated after first node event. |
| | `result` | `Optional<RunResult>` | yes | Populated after `RunCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `RunFailed` / `RunFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `RunHaltedOperator` / `RunHaltedAutomatic`. |
| | `createdAt` | `Instant` | no | When `RunCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the run reached a terminal status. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared on `HaltCleared`. |

## Enums

- `NodeExecutionVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `SANITIZED`.
- `RunStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`GraphRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunCreated` | `runId, graphJson, createdAt` | → PLANNING |
| `RunPlanned` | `plan: GraphPlan` | → EXECUTING |
| `NodeDispatched` | `nodeId, toolAnnotation` | no status change |
| `NodeBlocked` | `nodeId, attempt, blocker` | no status change; appends `TraceEntry` with `BLOCKED_BY_GUARDRAIL` |
| `NodeExecuted` | `entry: TraceEntry, updatedFields: Map<String, Object>` | no status change; appends entry, merges fields into `currentState` |
| `StateUpdated` | `fields: Map<String, Object>` | no status change; re-merge for explicit state corrections |
| `RunCompleted` | `result: RunResult` | → COMPLETED, `finishedAt = now` |
| `RunFailed` | `failureReason: String` | → FAILED, `finishedAt = now` |
| `RunHaltedOperator` | `haltReason: String` | → HALTED, `finishedAt = now` |
| `RunHaltedAutomatic` | `haltReason: String` | → HALTED, `finishedAt = now` |
| `RunFailedTimeout` | `failureReason: String` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RunQueueEntity`)

| Event | Payload |
|---|---|
| `RunSubmitted` | `runId, graphJson, requestedBy, submittedAt` |

## View row

`GraphRunRow` mirrors `GraphRun` minus the heavy trace payload — `trace` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each `scrubbedContent` is capped at 240 characters. The UI fetches the full run by id on click via `GET /api/runs/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
