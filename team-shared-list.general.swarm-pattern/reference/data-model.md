# Data model — swarm-pattern

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `JobBrief` | `jobId` | `String` | no | Id assigned at submission. |
| | `title` | `String` | no | Short job name. |
| | `description` | `String` | no | Free-text brief. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `ItemSpec` | `title` | `String` | no | Short imperative item title. |
| | `description` | `String` | no | One or two sentences a worker can act on. |
| | `dependsOn` | `List<String>` | no | Titles of items in the same plan that must finish first (may be empty). |
| `WorkPlan` | `planSummary` | `String` | no | One-sentence approach from the coordinator. |
| | `items` | `List<ItemSpec>` | no | Three to five item specs. |
| `OutputEntry` | `key` | `String` | no | Concise label for the output. |
| | `value` | `String` | no | Substantive result content (no placeholder stubs). |
| `RelayRequest` | `toWorker` | `String` | no | Worker id the request is addressed to. |
| | `question` | `String` | no | Concrete coordination question. |
| `WorkResult` | `itemId` | `String` | no | The item this result completes. |
| | `outputs` | `List<OutputEntry>` | no | One to four entries (empty when a relay request is raised). |
| | `summary` | `String` | no | One-sentence description of the work. |
| | `relayRequest` | `Optional<RelayRequest>` | yes | Present only when the worker is blocked on a peer. |
| `QualityReport` | `passed` | `boolean` | no | Whether the result passed the quality gate. |
| | `issues` | `List<String>` | no | One string per failing entry/check (empty on pass). |
| | `ranAt` | `Instant` | no | When the gate ran. |
| `RelayMessage` | `messageId` | `String` | no | Unique id. |
| | `fromWorker` | `String` | no | Sender worker id. |
| | `toWorker` | `String` | no | Recipient worker id. |
| | `itemId` | `String` | no | The stalled item the message concerns. |
| | `question` | `String` | no | The coordination question. |
| | `sentAt` | `Instant` | no | When posted. |
| | `reply` | `Optional<String>` | yes | Populated on reply. |
| | `repliedAt` | `Optional<Instant>` | yes | Populated on reply. |

## Entity state — `WorkItem` (`WorkItemEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `itemId` | `String` | no | Deterministic id `jobId + "-i" + index`. |
| `jobId` | `String` | no | Owning job. |
| `title` | `String` | no | From the `ItemSpec`. |
| `description` | `String` | no | From the `ItemSpec`. |
| `dependsOn` | `List<String>` | no | Dependency titles. |
| `status` | `WorkItemStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Worker that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `resultSummary` | `Optional<String>` | yes | Summary of the submitted result. |
| `outputCount` | `Optional<Integer>` | yes | Number of output entries in the result. |
| `qualityReport` | `Optional<QualityReport>` | yes | Latest quality-gate result. |
| `stalledReason` | `Optional<String>` | yes | Why the item is stalled. |
| `completedAt` | `Optional<Instant>` | yes | When the item reached `DONE`. |
| `createdAt` | `Instant` | no | When `WorkItemCreated` emitted. |

## Entity state — `Job` (`JobEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `jobId` | `String` | no | Unique id. |
| `title` | `String` | no | Job name. |
| `description` | `String` | no | Original brief. |
| `submittedBy` | `String` | no | Submitter id. |
| `status` | `JobStatus` | no | See enum. |
| `itemIds` | `List<String>` | no | Ids of items created from the plan. |
| `planSummary` | `Optional<String>` | yes | Populated on `JobPlanned`. |
| `createdAt` | `Instant` | no | When `JobCreated` emitted. |
| `finishedAt` | `Optional<Instant>` | yes | When the job reached `FINISHED`. |

## Enums

`WorkItemStatus`: `OPEN`, `CLAIMED`, `IN_PROGRESS`, `IN_REVIEW`, `DONE`, `STALLED`.
`JobStatus`: `PLANNING`, `READY`, `IN_PROGRESS`, `FINISHED`.

## Events — `WorkItemEntity`

| Event | Payload | Transition |
|---|---|---|
| `WorkItemCreated` | `itemId, jobId, title, description, dependsOn, createdAt` | → OPEN |
| `WorkItemClaimed` | `workerId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `WorkItemStarted` | `startedAt` | CLAIMED → IN_PROGRESS |
| `ResultSubmitted` | `summary, outputCount, submittedAt` | IN_PROGRESS → IN_REVIEW |
| `QualityPassed` | `qualityReport` | IN_REVIEW → DONE, `completedAt = now` |
| `QualityFailed` | `qualityReport` | IN_REVIEW → IN_PROGRESS (retry) |
| `WorkItemStalled` | `reason, stalledAt` | (any active) → STALLED |
| `WorkItemReleased` | `releasedAt` | CLAIMED/IN_PROGRESS → OPEN, clears `claimedBy` |
| `WorkItemCompleted` | `completedAt` | terminal marker emitted alongside `QualityPassed` |

Note: `QualityPassed` carries the `DONE` transition and `completedAt`; `WorkItemCompleted` is the explicit terminal event the job-completion check subscribes to. Prefer two events for a clean audit trail.

## Events — `JobEntity`

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, title, description, submittedBy, createdAt` | → PLANNING |
| `JobPlanned` | `itemIds, planSummary` | PLANNING → READY |
| `JobStarted` | `startedAt` | READY → IN_PROGRESS |
| `JobFinished` | `finishedAt` | IN_PROGRESS → FINISHED |

## Events — `RelayMailbox`

| Event | Payload |
|---|---|
| `RelayPosted` | `RelayMessage` |
| `RelayReplied` | `messageId, reply, repliedAt` |

## Events — `SubmissionLog`

| Event | Payload |
|---|---|
| `JobSubmitted` | `jobId, title, description, submittedBy, submittedAt` |

## Key-value state — `SwarmControl`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `paused` | `boolean` | no | Whether the swarm is frozen. |
| `pausedReason` | `Optional<String>` | yes | Operator-supplied reason. |
| `pausedBy` | `Optional<String>` | yes | Operator id. |
| `pausedAt` | `Optional<Instant>` | yes | When paused. |

## View row

`WorkItemRow` mirrors `WorkItem` but drops the full `OutputEntry` list — it keeps `resultSummary` and `outputCount` only, so the SSE stream stays small. Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6). The UI fetches nothing heavier; the output entries themselves are not projected (this sample stores only the summary on the entity).
