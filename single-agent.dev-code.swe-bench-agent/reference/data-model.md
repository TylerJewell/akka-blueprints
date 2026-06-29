# Data model — swe-bench-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BugDescription` | `taskId` | `String` | no | UUID minted by `BenchmarkEndpoint`. |
| | `title` | `String` | no | User-supplied label for the task. |
| | `description` | `String` | no | Narrative of the failure, including reproduction details. |
| | `language` | `String` | no | `"python"` / `"go"` / `"typescript"`. |
| | `repositoryName` | `String` | no | Logical name of the affected repository. |
| `PreparedSnapshot` | `snapshotRef` | `String` | no | SHA-256 hex of the normalized snapshot content. |
| | `filePaths` | `List<String>` | no | Relative paths of all files in the normalized snapshot. |
| | `testFilePaths` | `List<String>` | no | Subset of `filePaths` that are test files. |
| `HunkChange` | `filePath` | `String` | no | Relative path (no leading `/`). MUST match a `PreparedSnapshot.filePath`. |
| | `startLine` | `int` | no | 1-indexed first line of the changed range in the original file. |
| | `endLine` | `int` | no | 1-indexed last line of the changed range in the original file. |
| | `oldContent` | `String` | no | Original lines being replaced (verbatim). |
| | `newContent` | `String` | no | Replacement lines. |
| `PatchResult` | `unifiedDiff` | `String` | no | Well-formed unified diff; begins with `---`. |
| | `hunks` | `List<HunkChange>` | no | Structured representation of each changed range. |
| | `confidenceScore` | `int` | no | 0–100 inclusive. |
| | `patchSummary` | `String` | no | One sentence describing the fix. |
| | `patchedAt` | `Instant` | no | When the agent returned. |
| `TestCase` | `testId` | `String` | no | Unique identifier of the test stub. |
| | `testFile` | `String` | no | Relative path of the test file. |
| | `outcome` | `TestOutcome` | no | Enum value. |
| | `failureMessage` | `String` | yes | Non-null only when `outcome != PASS`. |
| `GateReport` | `verdict` | `GateVerdict` | no | Enum value. |
| | `testCases` | `List<TestCase>` | no | One entry per stub executed. |
| | `totalTests` | `int` | no | Count of stubs run. |
| | `passedTests` | `int` | no | Count of PASS outcomes. |
| | `runtimeMs` | `long` | no | Wall-clock milliseconds for the full gate run. |
| | `completedAt` | `Instant` | no | When `TestGateRunner` finished. |
| `BenchmarkTask` (entity state) | `taskId` | `String` | no | — |
| | `description` | `Optional<BugDescription>` | yes | Populated after `TaskSubmitted`. |
| | `snapshot` | `Optional<PreparedSnapshot>` | yes | Populated after `SnapshotPrepared`. |
| | `patch` | `Optional<PatchResult>` | yes | Populated after `PatchProduced`. |
| | `gateReport` | `Optional<GateReport>` | yes | Populated after `GateReportRecorded`. |
| | `status` | `TaskStatus` | no | See enum. |
| | `attemptNumber` | `int` | no | Increments on each `patchStep` retry. |
| | `createdAt` | `Instant` | no | When `TaskSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `BenchmarkTask` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TestOutcome`: `PASS`, `FAIL`, `ERROR`.
`GateVerdict`: `PASS`, `FAIL`.
`TaskStatus`: `SUBMITTED`, `SNAPSHOT_PREPARED`, `PATCHING`, `PATCH_PRODUCED`, `GATE_PASSED`, `GATE_FAILED`, `COMPLETED`, `FAILED`.

## Events (`BenchmarkTaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskSubmitted` | `description` | → SUBMITTED |
| `SnapshotPrepared` | `snapshot` | → SNAPSHOT_PREPARED |
| `PatchingStarted` | — | → PATCHING |
| `PatchProduced` | `patch` | → PATCH_PRODUCED |
| `GateReportRecorded` | `report` | → GATE_PASSED or GATE_FAILED (based on `report.verdict`) |
| `TaskCompleted` | — | → COMPLETED (terminal happy) |
| `TaskFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `BenchmarkTask.initial("")` with all `Optional` fields as `Optional.empty()`, `status = SUBMITTED`, and `attemptNumber = 0`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BenchmarkRow` mirrors `BenchmarkTask` minus the raw snapshot bytes (the audit log keeps the `snapshotRef`; the view holds `PreparedSnapshot.filePaths` and `testFilePaths` for the UI). The UI accesses snapshot content on demand via `GET /api/tasks/{id}`.

The view declares ONE query: `getAllTasks: SELECT * AS tasks FROM benchmark_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`BenchmarkTasks.java`)

```java
public final class BenchmarkTasks {
  public static final Task<PatchResult> PATCH_ISSUE = Task
      .name("Patch issue")
      .description("Read the attached repository snapshot and produce a PatchResult fixing the described bug")
      .resultConformsTo(PatchResult.class);

  private BenchmarkTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
