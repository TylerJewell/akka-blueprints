# Data model — code-interpreter-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `JobRequest` | `jobId` | `String` | no | UUID minted by `JobEndpoint`. |
| | `prompt` | `String` | no | User's natural-language question. |
| | `dataPayload` | `String` | no | Raw data supplied by the user (CSV / JSON / numeric text). |
| | `dataFormat` | `DataFormat` | no | Enum value declared by the user. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `GeneratedCode` | `pythonSource` | `String` | no | Python code produced by the agent. |
| | `importsList` | `List<String>` | no | Module names extracted from import statements at guard time. |
| | `generatedAt` | `Instant` | no | When the agent emitted the tool call. |
| `SandboxResult` | `status` | `SandboxStatus` | no | Enum value. |
| | `answer` | `String` | no | Scalar answer string; empty for tabular output. |
| | `rows` | `List<List<String>>` | no | Table rows; empty for scalar output. |
| | `columnNames` | `List<String>` | no | Column headers; empty for scalar output. |
| | `errorMessage` | `String` | no | Non-empty on `RUNTIME_ERROR`. |
| | `wallClockMillis` | `long` | no | Elapsed execution time in milliseconds. |
| `ExecutionResult` | `kind` | `ResultKind` | no | Enum value. |
| | `answer` | `String` | no | Scalar answer; empty for `TABLE` or `ERROR`. |
| | `rows` | `List<List<String>>` | no | Table rows; empty for `SCALAR` or `ERROR`. |
| | `columnNames` | `List<String>` | no | Column headers; empty for `SCALAR` or `ERROR`. |
| | `pythonSourceRan` | `String` | no | The approved code that actually executed. |
| | `decidedAt` | `Instant` | no | When the sandbox returned. |
| `Job` (entity state) | `jobId` | `String` | no | — |
| | `request` | `Optional<JobRequest>` | yes | Populated after `JobSubmitted`. |
| | `generatedCode` | `Optional<GeneratedCode>` | yes | Populated after `CodeApproved`. |
| | `result` | `Optional<ExecutionResult>` | yes | Populated after `ResultRecorded`. |
| | `status` | `JobStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `JobSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Job` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DataFormat`: `CSV`, `JSON`, `NUMERIC`.
`SandboxStatus`: `COMPLETED`, `TIMED_OUT`, `MEMORY_EXCEEDED`, `RUNTIME_ERROR`.
`ResultKind`: `SCALAR`, `TABLE`, `ERROR`.
`JobStatus`: `SUBMITTED`, `CODE_APPROVED`, `EXECUTING`, `RESULT_RECORDED`, `HALTED`, `FAILED`.

## Events (`JobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobSubmitted` | `request` | → SUBMITTED |
| `CodeApproved` | `generatedCode` | → CODE_APPROVED |
| `ExecutionStarted` | — | → EXECUTING |
| `ResultRecorded` | `result` | → RESULT_RECORDED (terminal happy) |
| `JobHalted` | `reason: String, breachType: String, wallClockMillis: long` | → HALTED (terminal) |
| `JobFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Job.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`JobRow` mirrors `Job` minus `request.dataPayload` (the payload can be large; omitting it keeps the view row compact for the list API). The UI fetches the full record on demand via `GET /api/jobs/{id}` and reads `request.dataPayload` from the JSON.

The view declares ONE query: `getAllJobs: SELECT * AS jobs FROM job_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`InterpretationTasks.java`)

```java
public final class InterpretationTasks {
  public static final Task<ExecutionResult> INTERPRET_DATA = Task
      .name("Interpret data")
      .description("Generate Python code to answer the prompt and return an ExecutionResult")
      .resultConformsTo(ExecutionResult.class);

  private InterpretationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
