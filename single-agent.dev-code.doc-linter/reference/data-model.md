# Data model — doc-linter

Every Java record the generated system defines. Nullable lifecycle fields are `Optional<T>`
(Lesson 6); Akka serializes `Optional<T>` as the raw value or `null` on the wire.

## Records

### LintRun (LintEntity state + LintView row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Run id (UUID), equals the entity id. |
| `filePath` | `String` | no | Requested file path. |
| `status` | `LintStatus` | no | Lifecycle state. |
| `requestedAt` | `Instant` | no | When the run was created. |
| `lintedAt` | `Optional<Instant>` | yes | Set when findings are recorded. |
| `findings` | `List<Finding>` | no | Empty until linted; never null. |
| `summary` | `Optional<String>` | yes | Agent's prose summary. |
| `gate` | `Optional<GateStatus>` | yes | PASS/FAIL once linted. |
| `feedbackNote` | `Optional<String>` | yes | Last feedback note. |
| `feedbackAt` | `Optional<Instant>` | yes | When feedback was recorded. |
| `blockReason` | `Optional<String>` | yes | Set when status is BLOCKED. |

`emptyState()` returns `LintRun.initial("")` with placeholder values and no
`commandContext()` reference (Lesson 3).

### Finding

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `rule` | `String` | no | Lint rule id (e.g. `no-h1`, `line-length`). |
| `line` | `int` | no | 1-based line number. |
| `severity` | `Severity` | no | ERROR / WARNING / INFO. |
| `message` | `String` | no | Human-readable description. |

### RuleProfile (RuleProfileEntity state)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `weights` | `Map<String,Integer>` | no | Rule id → weight; default 1. |

### LintCommand (LinterAgent input)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `runId` | `String` | no | Target run id. |
| `filePath` | `String` | no | File to lint. |
| `profile` | `Map<String,Integer>` | no | Current rule weights. |

### LintResult (LinterAgent output)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `orderedFindingRules` | `List<String>` | no | Rules ranked by priority. |
| `summary` | `String` | no | Prose summary of edits. |

## Enums

- `LintStatus`: `REQUESTED, LINTED, FEEDBACK_APPLIED, BLOCKED`.
- `GateStatus`: `PASS, FAIL`.
- `Severity`: `ERROR, WARNING, INFO`.

## Events (LintEntity)

| Event | Trigger |
|---|---|
| `LintRequested` | `requestLint(filePath)` command. |
| `LintCompleted` | `recordResult(findings, summary, gate)` after the agent returns. |
| `FeedbackRecorded` | `recordFeedback(rule, note)` from the feedback endpoint. |
| `LintBlocked` | `markBlocked(reason)` when the guardrail rejects the path. |

## View row

`LintView` rows are `LintRun` records. One query: `getAllRuns` —
`SELECT * AS runs FROM lint_view`. No `WHERE` on the `status` enum (Akka cannot auto-index
enum columns, Lesson 2); filter by status client-side in callers.
