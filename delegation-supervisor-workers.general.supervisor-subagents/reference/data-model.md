# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## TaskRecord (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `taskId` | `String` | no | UUID, also the workflow id |
| `description` | `String` | no | The submitted task description |
| `status` | `TaskStatus` | no | Lifecycle state |
| `routingPlan` | `Optional<RoutingPlan>` | yes | Supervisor routing decision; null until `TaskRouted` |
| `data` | `Optional<DataBundle>` | yes | DataSubagent output; null until `SubagentDataReturned` |
| `summary` | `Optional<SummaryOutput>` | yes | SummarySubagent output; null until `SubagentSummaryReturned` |
| `result` | `Optional<TaskResult>` | yes | Assembled result; null until `TaskCompleted` or `TaskDegraded` |
| `failureReason` | `Optional<String>` | yes | Set on `TaskBlocked` or `TaskDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `RoutingEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line routing-accuracy justification |
| `createdAt` | `Instant` | no | Task creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `TaskView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TaskRequest(String description, String submittedBy) {}

record RoutingPlan(String dataQuery, String summaryPrompt) {}

record DataItem(String label, String value, String source) {}
record DataBundle(List<DataItem> items, Instant retrievedAt) {}

record SummaryOutput(String narrative, List<String> keyPoints, Instant generatedAt) {}

record TaskResult(
    String narrative,
    DataBundle data,
    SummaryOutput summary,
    String routingRationale,
    Instant assembledAt
) {}
```

## Status enum

```java
enum TaskStatus { RECEIVED, ROUTING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED }
```

## Events

### TaskEntity

| Event | Trigger |
|---|---|
| `TaskReceived` | Workflow creates the task (`createTask`) |
| `TaskRouted` | Supervisor returns a `RoutingPlan` and guardrail passes |
| `SubagentDataReturned` | DataSubagent returns a `DataBundle` |
| `SubagentSummaryReturned` | SummarySubagent returns a `SummaryOutput` |
| `TaskCompleted` | Supervisor assembly succeeds |
| `TaskDegraded` | A subagent timed out; assembled from partial input |
| `TaskBlocked` | Routing guardrail rejected the `RoutingPlan` |
| `RoutingEvalScored` | `RoutingEvalSampler` recorded a 1–5 routing-accuracy score |

### TaskQueue

| Event | Trigger |
|---|---|
| `TaskSubmitted` | `submitTask(description, submittedBy)` from endpoint or simulator |

Fields: `{ taskId, description, submittedBy, submittedAt }`.
