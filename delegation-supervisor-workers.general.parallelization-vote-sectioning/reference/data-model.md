# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## Job (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `jobId` | `String` | no | UUID, also the workflow id |
| `prompt` | `String` | no | The submitted task prompt |
| `mode` | `ParallelMode` | no | SECTIONING or VOTING |
| `status` | `JobStatus` | no | Lifecycle state |
| `workerCount` | `int` | no | Number of workers dispatched; set after PARTITION |
| `sectionResults` | `Optional<List<SectionResult>>` | yes | Section worker outputs; null until WorkersDispatched; populated as results arrive |
| `voteResults` | `Optional<List<VoteResult>>` | yes | Vote worker outputs; null until WorkersDispatched; populated as results arrive |
| `aggregated` | `Optional<AggregatedOutput>` | yes | Aggregated output; null until JobAggregated |
| `failureReason` | `Optional<String>` | yes | Set on JobBlocked |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until EvalScored |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Job creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `JobView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
enum ParallelMode { SECTIONING, VOTING }

record JobRequest(String prompt, ParallelMode mode, String submittedBy) {}

record SectionTask(int sectionIndex, int totalSections, String sectionPrompt) {}
record SectionResult(int sectionIndex, String content, Instant processedAt) {}

record VotePrompt(String prompt, int workerIndex, int totalWorkers) {}
record VoteResult(int workerIndex, String answer, double confidence, Instant votedAt) {}

record Partition(List<SectionTask> sections, int workerCount) {}
record VotePlan(VotePrompt sharedPrompt, int workerCount) {}

record AggregatedOutput(
    String answer,
    ParallelMode mode,
    List<SectionResult> sectionOutputs,
    List<VoteResult> voteOutputs,
    String aggregationVerdict,
    Instant aggregatedAt
) {}
```

## Status enums

```java
enum JobStatus   { PARTITIONING, IN_PROGRESS, AGGREGATED, PARTIAL, BLOCKED }
enum ParallelMode { SECTIONING, VOTING }
```

## Events

### JobEntity

| Event | Trigger |
|---|---|
| `JobCreated` | Workflow creates the job (`createJob`) |
| `WorkersDispatched` | Supervisor returns the partition plan; workers are dispatched |
| `SectionResultReceived` | A SectionWorker returns a `SectionResult` |
| `VoteResultReceived` | A VoteWorker returns a `VoteResult` |
| `JobAggregated` | Supervisor aggregation passes the guardrail |
| `JobPartial` | At least one worker timed out; aggregated from partial results |
| `JobBlocked` | Guardrail rejected the aggregated output |
| `EvalScored` | `EvalSampler` recorded a 1–5 score |

### JobQueue

| Event | Trigger |
|---|---|
| `JobSubmitted` | `enqueueJob(prompt, mode, submittedBy)` from endpoint or simulator |

Fields: `{ jobId, prompt, mode, submittedBy, submittedAt }`.
