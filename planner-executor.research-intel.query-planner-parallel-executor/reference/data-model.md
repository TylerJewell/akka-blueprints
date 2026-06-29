# Data model — query-planner-parallel-executor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `question` | `String` | no | User-submitted research question. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `SubQuery` | `subQueryId` | `String` | no | Unique id within the plan (e.g., `sq-1`). |
| | `queryText` | `String` | no | One-sentence retrieval query. |
| | `strategy` | `RetrievalStrategy` | no | `CORPUS`, `WEB`, or `KNOWLEDGE_BASE`. |
| | `round` | `int` | no | 1-based round number in which this sub-query was issued. |
| `QueryPlan` | `subQueries` | `List<SubQuery>` | no | All sub-queries for the current round. |
| | `coverageGoal` | `String` | no | One-sentence description of what constitutes sufficient coverage. |
| | `round` | `int` | no | Current round number (1-based). |
| | `revisionReason` | `Optional<String>` | yes | Populated when the plan is a revision of a rejected plan. |
| `SubQueryResult` | `subQueryId` | `String` | no | Matches a `SubQuery.subQueryId` in the current plan. |
| | `strategy` | `RetrievalStrategy` | no | Strategy the executor used. |
| | `queryText` | `String` | no | Echo of the sub-query text. |
| | `ok` | `boolean` | no | True if the executor could fulfill the sub-query. |
| | `content` | `String` | no | Scrubbed textual result. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| | `verdict` | `ResultVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / TIMED_OUT. |
| | `retrievedAt` | `Instant` | no | When the result was collected. |
| `PlanEvaluation` | `score` | `double` | no | 0.0–1.0 composite plan-quality score. |
| | `passing` | `boolean` | no | True if `score >= 0.5`. |
| | `rationale` | `String` | no | One-sentence explanation of the score. |
| | `evaluatedAt` | `Instant` | no | When the evaluation ran. |
| `ResearchAnswer` | `summary` | `String` | no | 80–150 word synthesised answer. |
| | `citations` | `List<String>` | no | 3–6 cited sub-queries, each tagged by strategy. |
| | `confidence` | `double` | no | 0.0–1.0 answer confidence. |
| | `roundsCompleted` | `int` | no | Number of retrieval rounds that ran. |
| | `producedAt` | `Instant` | no | When the synthesis agent produced the answer. |
| `QuerySession` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `question` | `String` | no | Original research question. |
| | `status` | `SessionStatus` | no | See enum. |
| | `currentPlan` | `Optional<QueryPlan>` | yes | Populated after `SessionPlanned`; updated on revision. |
| | `lastEval` | `Optional<PlanEvaluation>` | yes | Populated after `PlanEvaluated`. |
| | `results` | `List<SubQueryResult>` | no | Append-only; grows across all rounds. Empty before first `SubQueryRecorded`. |
| | `answer` | `Optional<ResearchAnswer>` | yes | Populated after `SessionCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `SessionFailed` / `SessionTimedOut`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `SessionHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `CoverageDecision` | (sealed interface) | — | — | Permits `Sufficient`, `NeedsFollowup(QueryPlan revised)`, `Fail(String reason)`. |

## Enums

- `RetrievalStrategy` → `CORPUS`, `WEB`, `KNOWLEDGE_BASE`.
- `ResultVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `TIMED_OUT`.
- `SessionStatus` → `PLANNING`, `EVALUATING`, `EXECUTING`, `SYNTHESIZING`, `COMPLETED`, `FAILED`, `HALTED`, `STALE`.
- `CoverageDecision` is a sealed interface (not an enum) because each variant carries a payload.

## Events (`QuerySessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `sessionId, question, createdAt` | → PLANNING |
| `SessionPlanned` | `plan: QueryPlan` | → EVALUATING |
| `PlanEvaluated` | `eval: PlanEvaluation` | stays EVALUATING or → EXECUTING on passing |
| `PlanRevisionRequested` | `reason: String` | no status change; logged on entity |
| `SubQueryDispatched` | `subQueryId, strategy` | no status change; marks sub-query in-flight |
| `SubQueryBlocked` | `subQueryId, blocker: String` | no status change; records a BLOCKED_BY_GUARDRAIL result |
| `SubQueryRecorded` | `result: SubQueryResult` | no status change; appends to `results` |
| `CoverageAssessed` | `decision: CoverageDecision` | → EVALUATING (NeedsFollowup), SYNTHESIZING (Sufficient), FAILED (Fail) |
| `SynthesisStarted` | `sessionId, roundsCompleted` | no status change |
| `SessionCompleted` | `answer: ResearchAnswer` | → COMPLETED, `finishedAt = now` |
| `SessionFailed` | `failureReason: String` | → FAILED, `finishedAt = now` |
| `SessionHaltedOperator` | `haltReason: String` | → HALTED, `finishedAt = now` |
| `SessionTimedOut` | `failureReason: String` | → STALE, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason: String, haltedAt: Instant` |
| `HaltCleared` | `clearedAt: Instant` |

## Events (`QueryQueue`)

| Event | Payload |
|---|---|
| `QuerySubmitted` | `sessionId, question, requestedBy, submittedAt` |

## View row

`SessionRow` mirrors `QuerySession` but `results` is truncated to the last 5 `SubQueryResult` entries plus a `truncatedFromTotal: int` count. Each entry's `content` is capped at 240 characters. The UI fetches the full session by id on click via `GET /api/sessions/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
