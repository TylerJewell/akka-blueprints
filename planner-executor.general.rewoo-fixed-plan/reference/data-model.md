# Data model — rewoo-fixed-plan

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `query` | `String` | no | User-submitted query text. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `PlanStep` | `stepIndex` | `int` | no | 0-based index; determines execution order. |
| | `tool` | `ToolKind` | no | Which tool simulation to invoke. |
| | `inputExpression` | `String` | no | Tool input, possibly containing `#En` references. |
| | `result` | `Optional<String>` | yes | Null in the original plan; populated after `StepCompleted`. |
| `QueryPlan` | `steps` | `List<PlanStep>` | no | Ordered list of plan steps (3–5). Immutable after `QueryPlanned`. |
| `ToolResult` | `tool` | `ToolKind` | no | Tool kind that was called. |
| | `resolvedInput` | `String` | no | The `inputExpression` after `#En` substitution. |
| | `ok` | `boolean` | no | True if the tool simulation returned a usable result. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `StepRecord` | `stepIndex` | `int` | no | Index of the plan step this record covers. |
| | `tool` | `ToolKind` | no | Tool kind executed (or blocked). |
| | `resolvedInput` | `String` | no | Input used for the actual tool call. |
| | `verdict` | `StepVerdict` | no | COMPLETED / BLOCKED_BY_GUARDRAIL / FAILED. |
| | `scrubbedResult` | `String` | no | Sanitized result text. Empty string on non-COMPLETED verdicts. |
| | `blockReason` | `Optional<String>` | yes | Populated when `verdict = BLOCKED_BY_GUARDRAIL` or `FAILED`. |
| | `recordedAt` | `Instant` | no | When the record was appended. |
| `SolvedAnswer` | `summary` | `String` | no | 60–120 word synthesis of the filled plan results. |
| | `citations` | `List<String>` | no | 3–5 bullets, each referencing a step index and tool kind. |
| | `producedAt` | `Instant` | no | When the Solver produced the answer. |
| `Query` (entity state) | `queryId` | `String` | no | Unique id. |
| | `query` | `String` | no | Original query text. |
| | `status` | `QueryStatus` | no | See enum. |
| | `plan` | `Optional<QueryPlan>` | yes | Populated after `QueryPlanned`; steps gain `result` values progressively. |
| | `stepRecords` | `List<StepRecord>` | no | Append-only; one entry per executed or blocked step. Empty until first `StepCompleted` or `StepBlocked`. |
| | `answer` | `Optional<SolvedAnswer>` | yes | Populated after `QuerySolved`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `QueryFailed` / `QueryFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `QueryHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `QueryCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the query reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |

## Enums

- `ToolKind` → `WEB_SEARCH`, `FILE_READ`, `CALCULATE`, `CODE_EVAL`.
- `StepVerdict` → `COMPLETED`, `BLOCKED_BY_GUARDRAIL`, `FAILED`.
- `QueryStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryCreated` | `queryId, query, createdAt` | → PLANNING |
| `QueryPlanned` | `plan: QueryPlan` | → EXECUTING |
| `StepStarted` | `stepIndex, tool, resolvedInput` | no status change; records that execution of this step has begun. |
| `StepBlocked` | `stepIndex, tool, resolvedInput, blockReason` | → FAILED, `finishedAt = now`; appends a `StepRecord` with `verdict = BLOCKED_BY_GUARDRAIL`. |
| `StepCompleted` | `stepRecord: StepRecord` | no status change; appends to `stepRecords` and fills `plan.steps[stepIndex].result`. |
| `QuerySolved` | `answer: SolvedAnswer` | → COMPLETED, `finishedAt = now` |
| `QueryFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `QueryHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `QueryFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `QuerySubmitted` | `queryId, query, requestedBy, submittedAt` |

## View row

`QueryRow` mirrors `Query` with the following truncations: `stepRecords` is limited to the last 3 entries plus a `truncatedFromTotal: int` count; each entry's `scrubbedResult` is capped at 240 characters; `plan.steps[n].result` values are truncated to 120 characters. The UI fetches the full query by id on click via `GET /api/queries/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
