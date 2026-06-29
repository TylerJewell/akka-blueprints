# Data model — codeact-sandboxed-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskDefinition` | `taskId` | `String` | no | UUID minted by `TaskEndpoint`. |
| | `description` | `String` | no | Natural-language problem statement. |
| | `contextData` | `String` | no | Input data for the code (piped to stdin); may be empty string. |
| | `acceptanceCriterion` | `String` | no | Rule the output must satisfy for the task to be `SOLVED`. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `CodeSnippet` | `iterationNumber` | `int` | no | 1-based index of this execution attempt. |
| | `language` | `String` | no | Always `"python"` in this blueprint's sandbox. |
| | `code` | `String` | no | The generated code that was (or would have been) executed. |
| | `generatedAt` | `Instant` | no | When the agent emitted the `execute_code` tool call. |
| `ExecutionResult` | `iterationNumber` | `int` | no | Matches the corresponding `CodeSnippet`. |
| | `rawOutput` | `String` | no | Unredacted stdout from the sandbox. Audit-only. |
| | `halted` | `boolean` | no | `true` when `SafetyHaltMonitor` fired on this output. |
| | `haltReason` | `String` | yes | Non-null only when `halted = true`. |
| | `executedAt` | `Instant` | no | When the sandbox returned. |
| `SanitizedOutput` | `iterationNumber` | `int` | no | Matches the corresponding `ExecutionResult`. |
| | `sanitizedOutput` | `String` | no | Secret-redacted stdout; this is what the UI and view expose. |
| | `secretCategoriesFound` | `List<String>` | no | e.g. `["aws-key","github-token"]`; empty list when nothing found. |
| `TaskResolution` | `status` | `ResolutionStatus` | no | Enum value. |
| | `finalOutput` | `String` | no | Accepted output on `SOLVED`; failure/halt description otherwise. |
| | `iterationsUsed` | `int` | no | Total `execute_code` calls made (including guardrail-rejected ones). |
| | `resolvedAt` | `Instant` | no | When the workflow reached a terminal step. |
| `Task` (entity state) | `taskId` | `String` | no | — |
| | `definition` | `Optional<TaskDefinition>` | yes | Populated after `TaskSubmitted`. |
| | `codeHistory` | `List<CodeSnippet>` | no | Accumulates one entry per iteration. Empty list initially. |
| | `outputHistory` | `List<SanitizedOutput>` | no | Accumulates one entry per sanitized iteration output. Empty list initially. |
| | `resolution` | `Optional<TaskResolution>` | yes | Populated after `TaskSolved` or `TaskFailed`. |
| | `status` | `TaskStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TaskSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Task` is `Optional<T>`. The `codeHistory` and `outputHistory` list fields are never `Optional` — they start as `List.of()` and each event-applier returns a new list with the appended entry. The view's table updater wraps Optional values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ResolutionStatus`: `SOLVED`, `FAILED`, `HALTED`.
`TaskStatus`: `SUBMITTED`, `EXECUTING`, `SOLVED`, `HALTED`, `FAILED`.

## Events (`TaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskSubmitted` | `definition` | → SUBMITTED |
| `CodeExecuted` | `snippet: CodeSnippet, rawOutput: String` | SUBMITTED → EXECUTING; EXECUTING → EXECUTING (each iteration) |
| `OutputSanitized` | `sanitizedOutput: SanitizedOutput` | no status change; appends to `outputHistory` |
| `HaltTriggered` | `reason: String` | EXECUTING → HALTED (terminal) |
| `TaskSolved` | `resolution: TaskResolution` | EXECUTING → SOLVED (terminal happy) |
| `TaskFailed` | `reason: String` | EXECUTING → FAILED (terminal) |

`emptyState()` returns `Task.initial("")` with `definition = Optional.empty()`, `codeHistory = List.of()`, `outputHistory = List.of()`, `resolution = Optional.empty()`, `status = SUBMITTED`, `finishedAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

Note: `CodeExecuted` is the only event that occurs more than once per entity. The applier appends `snippet` to `codeHistory`; it does NOT replace the list. `OutputSanitized` similarly appends to `outputHistory`. All other events occur at most once.

## View row

`TaskRow` mirrors `Task` minus `ExecutionResult.rawOutput` (the audit log keeps that; only operators with direct entity access can read the raw form via `GET /api/tasks/{id}` — the endpoint exposes the full `Task` including entity state, not the view row, for single-item fetches). The view exposes `codeHistory` (code snippets only) and `outputHistory` (sanitized outputs only).

The view declares ONE query: `getAllTasks: SELECT * AS tasks FROM task_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint sorts and filters client-side.

## Task definition (`CodeActTasks.java`)

```java
public final class CodeActTasks {
  public static final Task<TaskResolution> EXECUTE_CODE_TASK = Task
      .name("Execute code task")
      .description("Write and execute code to solve the submitted task; iterate on the output until the acceptance criterion is met")
      .resultConformsTo(TaskResolution.class);

  private CodeActTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Supporting-class contracts

**`SandboxGuardrail`** — implements `before-tool-call` hook interface. Input: tool name + arguments map. Output: `Guardrail.accept()` or `Guardrail.reject(structuredError)`. Stateless; same forbidden-pattern set for every call.

**`SafetyHaltMonitor`** — pure static method `Optional<String> inspect(String rawOutput)`. Returns `Optional.empty()` for clean output; `Optional.of(haltReason)` for matched output. Stateless; no LLM call.

**`AcceptanceChecker`** — pure static method `boolean accepted(TaskDefinition definition, SanitizedOutput output)`. For seeded tasks: exact-match or regex-match against `acceptanceCriterion`. For custom tasks: the deployer may replace this implementation with an LLM-based evaluator using the same interface. Stateless; no LLM call in the baseline implementation.
