# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## TaskRequest (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `requestId` | `String` | no | UUID, also the workflow id |
| `taskDescription` | `String` | no | The submitted task description |
| `status` | `TaskStatus` | no | Lifecycle state |
| `plan` | `Optional<ExecutionPlan>` | yes | Orchestrator's decomposition; null until `PlanAttached` |
| `writerOutput` | `Optional<WriterOutput>` | yes | WriterWorker output; null until `WriterOutputAttached` |
| `analyzerOutput` | `Optional<AnalyzerOutput>` | yes | AnalyzerWorker output; null until `AnalyzerOutputAttached` |
| `result` | `Optional<CompositeResult>` | yes | Synthesised result; null until `TaskCompleted` |
| `failureReason` | `Optional<String>` | yes | Set on `TaskBlocked` or `TaskDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `TaskEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Task creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `TaskView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TaskSubmission(String taskDescription, String requestedBy) {}
record SubTask(String subTaskId, String workerRole, String instruction) {}
record ExecutionPlan(String planId, List<SubTask> subTasks, String objective) {}
record WriterOutput(String content, List<String> sections, Instant producedAt) {}
record AnalyzerOutput(String evaluation, List<String> findings, Instant producedAt) {}
record CompositeResult(String summary, WriterOutput writerOutput, AnalyzerOutput analyzerOutput,
                       String guardrailVerdict, Instant completedAt) {}
```

## Status enum

```java
enum TaskStatus { PLANNING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED }
```

## Events

### TaskRequestEntity

| Event | Trigger |
|---|---|
| `TaskCreated` | Workflow creates the task (`createTask`) |
| `PlanAttached` | Orchestrator returns an `ExecutionPlan` and plan guardrail passes |
| `WriterOutputAttached` | WriterWorker returns a `WriterOutput` |
| `AnalyzerOutputAttached` | AnalyzerWorker returns an `AnalyzerOutput` |
| `TaskCompleted` | Orchestrator synthesis produces a `CompositeResult` |
| `TaskDegraded` | A worker timed out; synthesised from partial input |
| `TaskBlocked` | Plan guardrail rejected the `ExecutionPlan` |
| `TaskEvalScored` | `EvalSampler` recorded a 1–5 score |

### TaskQueue

| Event | Trigger |
|---|---|
| `TaskSubmitted` | `enqueueTask(taskDescription, requestedBy)` from endpoint or simulator |

Fields: `{ requestId, taskDescription, requestedBy, submittedAt }`.
