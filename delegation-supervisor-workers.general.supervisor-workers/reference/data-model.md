# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## CollaborationTask (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `taskId` | `String` | no | UUID, also the workflow id |
| `description` | `String` | no | The submitted task description |
| `status` | `TaskStatus` | no | Lifecycle state |
| `routingDecision` | `Optional<RoutingDecision>` | yes | Supervisor routing output; null until `TaskRouted` |
| `researchOutput` | `Optional<ResearchOutput>` | yes | ResearchWorker output; null until `ResearchOutputAttached` |
| `chartData` | `Optional<ChartData>` | yes | ChartWorker output; null until `ChartDataAttached` |
| `result` | `Optional<TaskResult>` | yes | Merged result; null until `TaskCompleted` |
| `failureReason` | `Optional<String>` | yes | Set on `TaskBlocked` or `TaskDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `TaskEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Task creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `TaskView` row type (`CollaborationTaskRow`) mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TaskRequest(String description, String requestedBy) {}
record RoutingDecision(String researchQuery, String chartDescription) {}
record ResearchFinding(String headline, String source, String content) {}
record ResearchOutput(List<ResearchFinding> findings, Instant gatheredAt) {}
record ChartSeries(String label, List<Double> values) {}
record ChartData(String chartType, String title, List<ChartSeries> series, Instant generatedAt) {}
record TaskResult(
    String summary,
    ResearchOutput researchOutput,
    ChartData chartData,
    String guardrailVerdict,
    Instant completedAt
) {}
```

## Status enum

```java
enum TaskStatus { ROUTING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED }
```

## Events

### TaskEntity

| Event | Trigger |
|---|---|
| `TaskCreated` | Workflow creates the task (`createTask`) |
| `TaskRouted` | Supervisor returns a `RoutingDecision` |
| `ResearchOutputAttached` | ResearchWorker returns a `ResearchOutput` |
| `ChartDataAttached` | ChartWorker returns a `ChartData` payload |
| `TaskCompleted` | Supervisor synthesis passes the guardrail |
| `TaskDegraded` | A worker timed out; synthesised from partial input |
| `TaskBlocked` | Guardrail rejected a disallowed tool call |
| `TaskEvalScored` | `EvalSampler` recorded a 1–5 score |

### TaskRequestQueue

| Event | Trigger |
|---|---|
| `TaskSubmitted` | `enqueueTask(description, requestedBy)` from endpoint or simulator |

Fields: `{ taskId, description, requestedBy, submittedAt }`.
