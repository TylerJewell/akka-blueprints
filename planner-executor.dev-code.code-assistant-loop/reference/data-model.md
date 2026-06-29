# Data model — code-assistant-loop

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskRequest` | `prompt` | `String` | no | Developer-submitted task description. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `PlanLedger` | `targetFiles` | `List<String>` | no | Absolute paths of files the planner intends to touch. |
| | `steps` | `List<String>` | no | Ordered list of planned actions (3–8). |
| | `currentDispatch` | `Optional<ActionDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `ActionDecision` | `actionKind` | `ActionKind` | no | Which action to perform. |
| | `targetFile` | `String` | no | Target file path. |
| | `instruction` | `String` | no | One-sentence instruction to the agent. |
| | `rationale` | `String` | no | One-sentence justification. |
| `ActionResult` | `actionKind` | `ActionKind` | no | Which action ran. |
| | `targetFile` | `String` | no | Target file path (echo of the decision). |
| | `ok` | `boolean` | no | True if the agent could fulfill the action. |
| | `content` | `String` | no | File content (READ_FILE) or change description (EDIT_FILE/RUN_TESTS). |
| | `diff` | `Optional<String>` | yes | Unified diff; populated for EDIT_FILE. |
| | `testOutput` | `Optional<String>` | yes | Full test-runner output; populated for RUN_TESTS. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `EditEntry` | `attempt` | `int` | no | 1-based attempt count for this `(actionKind, targetFile, instruction)` triple. |
| | `actionKind` | `ActionKind` | no | Which action ran. |
| | `targetFile` | `String` | no | Target file path. |
| | `instruction` | `String` | no | The instruction text. |
| | `verdict` | `EditVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / CI_GATE_FAILED / FAILED / UNSAFE. |
| | `content` | `String` | no | Agent-returned content. |
| | `diff` | `Optional<String>` | yes | Diff; populated for EDIT_FILE. |
| | `testOutput` | `Optional<String>` | yes | Test output; populated for RUN_TESTS. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `EditLog` | `entries` | `List<EditEntry>` | no | Append-only. |
| `CommitSummary` | `message` | `String` | no | Conventional-commit message. |
| | `filesChanged` | `List<String>` | no | Files with an EDIT_FILE OK entry. |
| | `testsPassed` | `boolean` | no | True only when the last RUN_TESTS verdict was OK. |
| | `producedAt` | `Instant` | no | When the planner produced the summary. |
| `Session` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original task prompt. |
| | `status` | `SessionStatus` | no | See enum. |
| | `plan` | `Optional<PlanLedger>` | yes | Populated after `SessionPlanned`. |
| | `editLog` | `Optional<EditLog>` | yes | Populated after first `EditRecorded` or `EditBlocked`. |
| | `commitSummary` | `Optional<CommitSummary>` | yes | Populated after `SessionCommitted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `SessionFailed` / `SessionFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `SessionHaltedOperator` / `SessionHaltedAutomatic`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextAction` | (sealed interface) | — | — | Permits `Continue(ActionDecision)`, `Replan(PlanLedger revised)`, `Commit(CommitSummary stub)`, `Fail(String reason)`. |
| `CiVerdict` | `passed` | `boolean` | no | Whether the test run passed. |
| | `totalTests` | `int` | no | Total tests reported. |
| | `failedTests` | `int` | no | Failed tests reported. |
| | `summary` | `String` | no | One-line human-readable summary. |

## Enums

- `ActionKind` → `READ_FILE`, `EDIT_FILE`, `RUN_TESTS`.
- `EditVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `CI_GATE_FAILED`, `FAILED`, `UNSAFE`.
- `SessionStatus` → `PLANNING`, `EXECUTING`, `COMMITTED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `sessionId, prompt, createdAt` | → PLANNING |
| `SessionPlanned` | `plan: PlanLedger` | → EXECUTING |
| `ActionDispatched` | `dispatch: ActionDecision` | no status change; sets `plan.currentDispatch`. |
| `EditBlocked` | `attempt, dispatch, blocker` | no status change; appends an `EditEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `CiGateFailed` | `entry: EditEntry` | no status change; appends an `EditEntry` with verdict `CI_GATE_FAILED`. |
| `EditRecorded` | `entry: EditEntry` | no status change; appends to `editLog.entries`. |
| `PlanRevised` | `plan: PlanLedger` | no status change; replaces `plan`. |
| `SessionCommitted` | `commitSummary` | → COMMITTED, `finishedAt = now` |
| `SessionFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `SessionHaltedAutomatic` | `haltReason` | → HALTED, `finishedAt = now` |
| `SessionHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `SessionFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`TaskQueue`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `sessionId, prompt, requestedBy, submittedAt` |

## View row

`SessionRow` mirrors `Session` minus the heavy edit-log payload — `editLog.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `content` is capped at 240 characters. The UI fetches the full session by id on click via `GET /api/sessions/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
