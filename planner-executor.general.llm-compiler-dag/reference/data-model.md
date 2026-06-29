# Data model — llm-compiler-dag

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `query` | `String` | no | User-submitted query text. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ToolCall` | `callId` | `String` | no | Unique node id within the plan (e.g., `c1`, `c2`). |
| | `tool` | `ToolKind` | no | Which tool type to invoke. |
| | `arguments` | `Map<String,String>` | no | Tool-specific arguments (e.g., `query`, `expression`, `key`, `code`). |
| | `dependsOn` | `List<String>` | no | `callId` values that must be resolved before this node may dispatch. Empty for independent nodes. |
| `CompilationPlan` | `description` | `String` | no | One-sentence summary of what the plan achieves. |
| | `toolCalls` | `List<ToolCall>` | no | Ordered list of DAG nodes (3–6). |
| `ToolResult` | `callId` | `String` | no | Echoes the `ToolCall.callId`. |
| | `tool` | `ToolKind` | no | Tool that produced this result. |
| | `ok` | `boolean` | no | True if the tool call succeeded. |
| | `output` | `String` | no | Scrubbed result text. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ResultSet` | `results` | `List<ToolResult>` | no | One entry per resolved `ToolCall`. |
| `QueryAnswer` | `summary` | `String` | no | 60–120 word synthesized answer. |
| | `citations` | `List<String>` | no | One bullet per contributing `ToolResult`, tagged by `callId` and tool kind. |
| | `producedAt` | `Instant` | no | When the Joiner produced the answer. |
| `Job` (entity state) | `jobId` | `String` | no | Unique id. |
| | `query` | `String` | no | Original query text. |
| | `status` | `JobStatus` | no | See enum. |
| | `plan` | `Optional<CompilationPlan>` | yes | Populated after `JobCompiled`. |
| | `results` | `Optional<ResultSet>` | yes | Populated after first `ToolCallResolved`. |
| | `answer` | `Optional<QueryAnswer>` | yes | Populated after `JobCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `JobFailed` / `JobFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `JobHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `JobCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |

## Enums

- `ToolKind` → `SEARCH`, `CALCULATOR`, `LOOKUP`, `CODE_EVAL`.
- `CallStatus` → `PENDING`, `RUNNING`, `RESOLVED`, `SKIPPED`, `FAILED`. (Used in UI display; the DAG node status is derived from recorded events.)
- `JobStatus` → `COMPILING`, `RUNNING`, `COMPLETED`, `FAILED`, `HALTED`, `STALE`.

## Events (`JobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, query, createdAt` | → COMPILING |
| `JobCompiled` | `plan: CompilationPlan` | → RUNNING |
| `ToolCallDispatched` | `callId, tool` | no status change; marks node as RUNNING in DAG. |
| `ToolCallBlocked` | `callId, tool, reason` | no status change; marks node as SKIPPED; adds to resolvedIds. |
| `ToolCallResolved` | `result: ToolResult` | no status change; adds `callId` to resolvedIds; appends to `results`. |
| `ToolCallFailed` | `callId, tool, errorReason` | no status change; increments error count. |
| `JobCompleted` | `answer: QueryAnswer` | → COMPLETED, `finishedAt = now` |
| `JobFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `JobHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `JobFailedTimeout` | `failureReason` | → STALE, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `QuerySubmitted` | `jobId, query, requestedBy, submittedAt` |

## View row

`JobRow` mirrors `Job` minus the heavy results payload — `results.results` is truncated to the last 3 `ToolResult` entries plus a `truncatedFromTotal: int` count, and each `output` is capped at 240 characters. The UI fetches the full job by id on click via `GET /api/jobs/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
