# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## TaskRequest (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `requestId` | `String` | no | UUID, also the workflow id |
| `inputText` | `String` | no | The submitted input text |
| `operation` | `String` | no | The requested operation (summarize / classify / translate) |
| `status` | `RequestStatus` | no | Lifecycle state |
| `routing` | `Optional<RoutingDecision>` | yes | Supervisor's routing decision; null until `RequestRouted` |
| `toolResult` | `Optional<ToolResult>` | yes | Sub-agent output; null until `ToolResultAttached` |
| `result` | `Optional<TaskResult>` | yes | Assembled final result; null until `RequestCompleted` |
| `failureReason` | `Optional<String>` | yes | Set on `RequestBlocked` or `RequestTimedOut` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `TaskEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Request creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `CompositionView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TaskSubmission(String inputText, String operation, String requestedBy) {}
record RoutingDecision(String selectedTool, String reasoning) {}
record ToolResult(String toolName, String output, Instant returnedAt) {}
record TaskResult(String operation, String toolUsed, String output,
                  String guardrailVerdict, Instant completedAt) {}
```

## Status enum

```java
enum RequestStatus { QUEUED, IN_PROGRESS, COMPLETED, BLOCKED, TIMED_OUT }
```

## Events

### TaskRequestEntity

| Event | Trigger |
|---|---|
| `RequestCreated` | Workflow creates the request (`createRequest`) |
| `RequestRouted` | Supervisor returns a `RoutingDecision` |
| `ToolResultAttached` | Selected sub-agent returns a `ToolResult` |
| `RequestCompleted` | Supervisor assembles a `TaskResult`; guardrail passed |
| `RequestBlocked` | Guardrail rejected the tool call before dispatch |
| `RequestTimedOut` | Sub-agent step did not respond within 60 s |
| `TaskEvalScored` | `EvalSampler` recorded a 1–5 score |

### TaskRequestQueue

| Event | Trigger |
|---|---|
| `TaskSubmitted` | `enqueueTask(inputText, operation, requestedBy)` from endpoint or simulator |

Fields: `{ requestId, inputText, operation, requestedBy, submittedAt }`.
