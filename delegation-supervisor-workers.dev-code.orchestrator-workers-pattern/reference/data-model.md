# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## EditTask (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `taskId` | `String` | no | UUID, also the workflow id |
| `description` | `String` | no | The submitted edit task description |
| `targetFiles` | `List<String>` | no | Declared file paths the task may edit |
| `status` | `TaskStatus` | no | Lifecycle state |
| `plan` | `Optional<DecompositionPlan>` | yes | Orchestrator's decomposition; null until `PlanReady` |
| `editedFiles` | `Optional<List<EditedFile>>` | yes | Worker outputs; null until first `FileEdited` |
| `reviews` | `Optional<List<ReviewVerdict>>` | yes | Reviewer outputs; null until first `FileReviewed` |
| `changeset` | `Optional<Changeset>` | yes | Merged output; null until `TaskCompleted` or `TaskDegraded` |
| `failureReason` | `Optional<String>` | yes | Set on `TaskBlocked` or `TaskDegraded` |
| `qualityScore` | `Optional<Integer>` | yes | 1–5; null until `QualityScored` |
| `qualityRationale` | `Optional<String>` | yes | One-line quality justification |
| `createdAt` | `Instant` | no | Task creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `EditTaskView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TaskRequest(String description, List<String> targetFiles, String requestedBy) {}

record FileInstruction(String filePath, String instruction) {}
record DecompositionPlan(List<FileInstruction> instructions, String objective) {}

record EditedFile(String filePath, String editedContent, String diffSummary) {}

record ReviewVerdict(String filePath, boolean approved, String feedback) {}

record Changeset(
    List<EditedFile> files,
    List<ReviewVerdict> reviews,
    String summary,
    String qualityVerdict,
    Instant completedAt
) {}
```

## Status enum

```java
enum TaskStatus { PLANNING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED }
```

## Events

### EditTaskEntity

| Event | Trigger |
|---|---|
| `TaskCreated` | Workflow creates the task (`createTask`) |
| `PlanReady` | Orchestrator returns a `DecompositionPlan` |
| `FileEdited` | A `FileEditor` worker returns an `EditedFile` |
| `FileReviewed` | An `EditReviewer` returns a `ReviewVerdict` |
| `TaskCompleted` | Orchestrator synthesis finishes and guardrail passes |
| `TaskDegraded` | A worker timed out; synthesised from partial input |
| `TaskBlocked` | Before-tool-call guardrail rejected a write |
| `QualityScored` | `QualitySampler` recorded a 1–5 score |

### EditJobQueue

| Event | Trigger |
|---|---|
| `TaskSubmitted` | `submitTask(description, targetFiles, requestedBy)` from endpoint or simulator |

Fields: `{ taskId, description, targetFiles, requestedBy, submittedAt }`.
