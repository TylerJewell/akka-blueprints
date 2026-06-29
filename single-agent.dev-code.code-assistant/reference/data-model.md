# Data model — code-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `FileSnapshot` | `filePath` | `String` | no | Relative file path within the repository (e.g., `src/main/java/io/example/App.java`). |
| | `content` | `String` | no | Full file content as text. Sent to the agent as a task attachment, not inline. |
| | `language` | `String` | no | Programming language identifier (e.g., `java`, `python`, `typescript`). |
| `RepoSnapshot` | `snapshotId` | `String` | no | Stable id for this snapshot; used for mock-response dispatch. |
| | `projectName` | `String` | no | Human-readable project label shown in the UI. |
| | `files` | `List<FileSnapshot>` | no | All files in the snapshot (1–N). |
| `TaskRequest` | `editId` | `String` | no | UUID minted by `EditEndpoint`. |
| | `taskDescription` | `String` | no | Plain-language description of what the developer wants changed. |
| | `repoSnapshot` | `RepoSnapshot` | no | The repository files the agent will read. |
| | `author` | `String` | no | Developer identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `FileChange` | `filePath` | `String` | no | Relative file path; must match a `FileSnapshot.filePath` in the submitted snapshot for MODIFY/DELETE. |
| | `changeType` | `ChangeType` | no | Enum value. |
| | `beforeContent` | `String` | no | Full file content before the change (empty string for ADD). |
| | `afterContent` | `String` | no | Full file content after the change (empty string for DELETE). |
| | `diffSummary` | `String` | no | One sentence describing what changed and why. |
| `EditPlan` | `confidence` | `Confidence` | no | Enum value. |
| | `rationale` | `String` | no | 1–3 sentences explaining the approach. |
| | `changes` | `List<FileChange>` | no | Proposed file changes. May be empty only if the agent cannot determine any change. |
| | `testCommand` | `String` | no | Runnable command to verify the proposed changes. |
| | `proposedAt` | `Instant` | no | When the agent returned the plan. |
| `GateResult` | `status` | `GateStatus` | no | Enum value. |
| | `testOutput` | `String` | no | Full stdout + stderr from the test command. |
| | `testsPassed` | `int` | no | Count of passing tests parsed from output. |
| | `testsFailed` | `int` | no | Count of failing tests parsed from output. |
| | `evaluatedAt` | `Instant` | no | When `CIGate` finished running the test command. |
| `Edit` (entity state) | `editId` | `String` | no | — |
| | `request` | `Optional<TaskRequest>` | yes | Populated after `TaskSubmitted`. |
| | `plan` | `Optional<EditPlan>` | yes | Populated after `EditProposed`. |
| | `gateResult` | `Optional<GateResult>` | yes | Populated after `GatePassed` or `GateFailed`. |
| | `status` | `EditStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TaskSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Edit` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ChangeType`: `ADD`, `MODIFY`, `DELETE`.
`Confidence`: `HIGH`, `MEDIUM`, `LOW`.
`GateStatus`: `PASSED`, `FAILED`.
`EditStatus`: `SUBMITTED`, `ANALYZING`, `EDIT_PROPOSED`, `GATE_PASSED`, `GATE_FAILED`, `FAILED`.

## Events (`EditEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskSubmitted` | `request` | → SUBMITTED |
| `AnalysisStarted` | — | → ANALYZING |
| `EditProposed` | `plan` | → EDIT_PROPOSED |
| `GatePassed` | `gateResult` | → GATE_PASSED (terminal happy) |
| `GateFailed` | `gateResult` | → GATE_FAILED (terminal gate-fail) |
| `EditFailed` | `reason: String` | → FAILED (terminal error) |

`emptyState()` returns `Edit.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`EditRow` mirrors `Edit`. The view row includes `request.repoSnapshot` for the UI's file list display. The agent never receives file content via the view — that path is only used for reading gate-result status; source code reaches the agent exclusively via `TaskDef.attachment(...)`.

The view declares ONE query: `getAllEdits: SELECT * AS edits FROM edit_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`EditTasks.java`)

```java
public final class EditTasks {
  public static final Task<EditPlan> PROPOSE_EDITS = Task
      .name("Propose edits")
      .description("Read the attached repository files and produce an EditPlan for the coding task")
      .resultConformsTo(EditPlan.class);

  private EditTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
