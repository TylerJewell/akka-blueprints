# Data model — custom-orchestration-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskRequest` | `prompt` | `String` | no | User-submitted task prompt. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `OrchestrationStrategy` | `name` | `String` | no | Unique strategy identifier. |
| | `description` | `String` | no | Human-readable purpose. |
| | `maxRouteIterations` | `int` | no | Maximum number of `CallTool` ticks allowed. |
| | `maxRevisits` | `int` | no | Maximum number of `Revisit` decisions allowed. |
| | `allowedTools` | `List<String>` | no | Tool names the orchestrator may call. |
| `ToolCall` | `toolName` | `String` | no | Name of the tool to invoke. |
| | `args` | `String` | no | Tool-specific argument string. |
| | `rationale` | `String` | no | One-sentence justification from the orchestrator. |
| `ToolResult` | `toolName` | `String` | no | Tool that ran the call. |
| | `ok` | `boolean` | no | True if the tool could fulfill the call. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `RoutingDecision` | `action` | `RoutingAction` | no | CALL_TOOL / REVISIT / CONCLUDE / ABORT. |
| | `toolCall` | `Optional<ToolCall>` | yes | Populated when `action=CALL_TOOL`. |
| | `revisitIndex` | `Optional<Integer>` | yes | Populated when `action=REVISIT`. |
| | `concludeAnswer` | `Optional<String>` | yes | Stub populated when `action=CONCLUDE`; replaced in `concludeStep`. |
| | `abortReason` | `Optional<String>` | yes | Populated when `action=ABORT`. |
| `TraceEntry` | `sequence` | `int` | no | 1-based position in the execution trace. |
| | `kind` | `TraceKind` | no | TOOL_CALL / REVISIT / BLOCKED. |
| | `toolCall` | `Optional<ToolCall>` | yes | Populated for TOOL_CALL and BLOCKED entries. |
| | `scrubbedResult` | `Optional<String>` | yes | Sanitized result text; absent for BLOCKED entries. |
| | `verdict` | `TraceVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED. |
| | `blocker` | `Optional<String>` | yes | Guardrail rejection reason; populated for BLOCKED entries. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ExecutionContext` | `goal` | `String` | no | The user's task goal. |
| | `strategy` | `OrchestrationStrategy` | no | Strategy snapshot held for the lifetime of the task. |
| | `traceEntries` | `List<TraceEntry>` | no | Append-only execution trace. |
| `TaskAnswer` | `summary` | `String` | no | 50–100 word summary. |
| | `citations` | `List<String>` | no | 2–4 cited strings, each tagged by tool name. |
| | `producedAt` | `Instant` | no | When the orchestrator produced the answer. |
| `Task` (entity state) | `taskId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original prompt. |
| | `status` | `TaskStatus` | no | See enum. |
| | `activeStrategy` | `Optional<OrchestrationStrategy>` | yes | Populated after `TaskInitialized`. |
| | `context` | `Optional<ExecutionContext>` | yes | Populated after `ContextInitialized`. |
| | `answer` | `Optional<TaskAnswer>` | yes | Populated after `TaskCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `TaskFailed` / `TaskFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `TaskHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `TaskCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the task reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `StrategyRegistryState` | `strategies` | `Map<String, OrchestrationStrategy>` | no | All registered strategies by name. |
| | `activeName` | `String` | no | The currently active strategy name. |

## Enums

- `RoutingAction` → `CALL_TOOL`, `REVISIT`, `CONCLUDE`, `ABORT`.
- `TraceKind` → `TOOL_CALL`, `REVISIT`, `BLOCKED`.
- `TraceVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`.
- `TaskStatus` → `INITIALIZING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`TaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `taskId, prompt, createdAt` | → INITIALIZING |
| `TaskInitialized` | `strategy: OrchestrationStrategy` | no status change; sets `activeStrategy` |
| `ContextInitialized` | `context: ExecutionContext` | → EXECUTING |
| `ToolDispatched` | `sequence, toolCall` | no status change; records pending dispatch |
| `ToolBlocked` | `sequence, toolCall, blocker` | no status change; appends a `TraceEntry { kind: BLOCKED, verdict: BLOCKED_BY_GUARDRAIL }` |
| `ToolRecorded` | `entry: TraceEntry` | no status change; appends to `context.traceEntries` |
| `ContextRevisited` | `sequence, revisitIndex` | no status change; appends a `TraceEntry { kind: REVISIT }` |
| `TaskCompleted` | `answer: TaskAnswer` | → COMPLETED, `finishedAt = now` |
| `TaskFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `TaskHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `TaskFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`StrategyRegistry`)

| Event | Payload |
|---|---|
| `StrategyRegistered` | `name, strategy, registeredAt` |
| `StrategyActivated` | `name, activatedAt` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `taskId, prompt, requestedBy, submittedAt` |

## View row

`TaskRow` mirrors `Task` minus the heavy trace payload — `context.traceEntries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedResult` is capped at 240 characters. The UI fetches the full task by id on click via `GET /api/tasks/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
