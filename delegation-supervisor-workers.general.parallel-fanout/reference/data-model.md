# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## TaskJob (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `jobId` | `String` | no | UUID, also the workflow id |
| `payload` | `String` | no | The submitted job description |
| `status` | `JobStatus` | no | Lifecycle state |
| `structural` | `Optional<StructuralOutput>` | yes | SubtaskWorkerA output; null until `StructuralOutputAttached` |
| `contextual` | `Optional<ContextualOutput>` | yes | SubtaskWorkerB output; null until `ContextualOutputAttached` |
| `consolidated` | `Optional<ConsolidatedResult>` | yes | Merged result; null until `JobConsolidated` |
| `failureReason` | `Optional<String>` | yes | Set on `JobRejected` or `JobDegraded` |
| `qualityScore` | `Optional<Integer>` | yes | 1–5; null until `QualityScored` |
| `qualityRationale` | `Optional<String>` | yes | One-line quality justification |
| `createdAt` | `Instant` | no | Job creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `TaskJobView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record JobRequest(String payload, String requestedBy) {}
record DecompositionPlan(String structuralQuery, String contextualQuery) {}
record StructuralElement(String label, String category, String detail) {}
record StructuralOutput(List<StructuralElement> elements, Instant processedAt) {}
record ContextualOutput(String enrichedContext, List<String> tags, Instant enrichedAt) {}
record ConsolidatedResult(String summary, StructuralOutput structural, ContextualOutput contextual,
                          String validationVerdict, Instant consolidatedAt) {}
```

## Status enum

```java
enum JobStatus { QUEUED, PROCESSING, CONSOLIDATED, DEGRADED, REJECTED }
```

## Events

### TaskJobEntity

| Event | Trigger |
|---|---|
| `JobCreated` | Workflow creates the job (`createJob`) |
| `StructuralOutputAttached` | SubtaskWorkerA returns a `StructuralOutput` |
| `ContextualOutputAttached` | SubtaskWorkerB returns a `ContextualOutput` |
| `JobConsolidated` | Coordinator consolidation passes validation |
| `JobDegraded` | A worker timed out; consolidated from partial input |
| `JobRejected` | Validation rejected the consolidated result |
| `QualityScored` | `QualitySampler` recorded a 1–5 score |

### JobQueue

| Event | Trigger |
|---|---|
| `JobSubmitted` | `submitJob(payload, requestedBy)` from endpoint or simulator |

Fields: `{ jobId, payload, requestedBy, submittedAt }`.
