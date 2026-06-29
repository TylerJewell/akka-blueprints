# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## GreetingRequest (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `requestId` | `String` | no | UUID, also the workflow id |
| `name` | `String` | no | The submitted person's name |
| `context` | `String` | no | The occasion or relationship context |
| `status` | `RequestStatus` | no | Lifecycle state |
| `draft` | `Optional<DraftMessage>` | yes | GreetingWriter output; null until `DraftAttached` |
| `followUp` | `Optional<FollowUpAction>` | yes | ActionAdvisor output; null until `FollowUpAttached` |
| `response` | `Optional<GreetingResponse>` | yes | Composed response; null until `RequestCompleted` |
| `failureReason` | `Optional<String>` | yes | Set on `RequestDegraded` if a worker timed out |
| `createdAt` | `Instant` | no | Request creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `GreetingView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record GreetingSubmission(String name, String context, String requestedBy) {}

record WritingSpec(String name, String context, String tone) {}
record AdvisorySpec(String name, String context) {}
record DecomposedWork(WritingSpec writingSpec, AdvisorySpec advisorySpec) {}

record DraftMessage(String message, Instant draftedAt) {}
record FollowUpAction(String action, Instant advisedAt) {}

record GreetingResponse(String message, String followUpAction, Instant composedAt) {}
```

## Status enum

```java
enum RequestStatus { PENDING, IN_PROGRESS, COMPLETED, DEGRADED }
```

## Events

### GreetingEntity

| Event | Trigger |
|---|---|
| `RequestCreated` | Workflow creates the entity (`createRequest`) |
| `DraftAttached` | GreetingWriter returns a `DraftMessage` |
| `FollowUpAttached` | ActionAdvisor returns a `FollowUpAction` |
| `RequestCompleted` | Coordinator composes the final response |
| `RequestDegraded` | A worker timed out; composed from partial input |

### RequestQueue

| Event | Trigger |
|---|---|
| `GreetingSubmitted` | `enqueueGreeting(name, context, requestedBy)` from endpoint or simulator |

Fields: `{ requestId, name, context, requestedBy, submittedAt }`.
