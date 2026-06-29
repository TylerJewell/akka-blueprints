# Data model — shared-task-team

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GoalBrief` | `goalId` | `String` | no | Id assigned at submission. |
| | `title` | `String` | no | Short goal name. |
| | `description` | `String` | no | Free-text brief. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `TaskSpec` | `title` | `String` | no | Short imperative task title. |
| | `description` | `String` | no | One or two sentences a worker can act on. |
| | `dependsOn` | `List<String>` | no | Titles of tasks in the same plan that must finish first (may be empty). |
| `TaskPlan` | `planSummary` | `String` | no | One-sentence approach from the goal planner. |
| | `tasks` | `List<TaskSpec>` | no | Three to six task specs. |
| `OutputSection` | `heading` | `String` | no | Short noun-phrase heading. |
| | `content` | `String` | no | Section body (no placeholder stubs). |
| `PeerRequest` | `toWorker` | `String` | no | Worker id the request is addressed to. |
| | `question` | `String` | no | Concrete coordination question. |
| `ResultArtifact` | `taskId` | `String` | no | The task this artifact completes. |
| | `sections` | `List<OutputSection>` | no | One to three sections (empty when a peer request is raised). |
| | `summary` | `String` | no | One-sentence description of the work. |
| | `peerRequest` | `Optional<PeerRequest>` | yes | Present only when the worker is blocked on a peer. |
| `QualityReport` | `passed` | `boolean` | no | Whether the artifact passed the quality check. |
| | `failures` | `List<String>` | no | One string per failing section/check (empty on pass). |
| | `checkedAt` | `Instant` | no | When the check ran. |
| `PeerMessage` | `messageId` | `String` | no | Unique id. |
| | `fromWorker` | `String` | no | Sender worker id. |
| | `toWorker` | `String` | no | Recipient worker id. |
| | `taskId` | `String` | no | The blocked task the message concerns. |
| | `question` | `String` | no | The coordination question. |
| | `sentAt` | `Instant` | no | When posted. |
| | `reply` | `Optional<String>` | yes | Populated on reply. |
| | `repliedAt` | `Optional<Instant>` | yes | Populated on reply. |

## Entity state — `Task` (`TaskEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `taskId` | `String` | no | Deterministic id `goalId + "-t" + index`. |
| `goalId` | `String` | no | Owning goal. |
| `title` | `String` | no | From the `TaskSpec`. |
| `description` | `String` | no | From the `TaskSpec`. |
| `dependsOn` | `List<String>` | no | Dependency titles. |
| `status` | `TaskStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Worker that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `artifactSummary` | `Optional<String>` | yes | Summary of the submitted artifact. |
| `sectionCount` | `Optional<Integer>` | yes | Number of sections in the artifact. |
| `qualityReport` | `Optional<QualityReport>` | yes | Latest quality-check result. |
| `blockedReason` | `Optional<String>` | yes | Why the task is blocked. |
| `completedAt` | `Optional<Instant>` | yes | When the task reached `DONE`. |
| `createdAt` | `Instant` | no | When `TaskCreated` emitted. |

## Entity state — `Goal` (`GoalEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `goalId` | `String` | no | Unique id. |
| `title` | `String` | no | Goal name. |
| `description` | `String` | no | Original brief. |
| `requestedBy` | `String` | no | Submitter id. |
| `status` | `GoalStatus` | no | See enum. |
| `taskIds` | `List<String>` | no | Ids of tasks created from the plan. |
| `planSummary` | `Optional<String>` | yes | Populated on `GoalPlanned`. |
| `createdAt` | `Instant` | no | When `GoalCreated` emitted. |
| `completedAt` | `Optional<Instant>` | yes | When the goal reached `COMPLETED`. |
| `evalResult` | `Optional<EvalResult>` | yes | On-decision eval outcome, recorded after completion. |

## Enums

`TaskStatus`: `OPEN`, `CLAIMED`, `IN_PROGRESS`, `IN_REVIEW`, `DONE`, `BLOCKED`.
`GoalStatus`: `PLANNING`, `READY`, `IN_PROGRESS`, `COMPLETED`.

## Events — `TaskEntity`

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `taskId, goalId, title, description, dependsOn, createdAt` | → OPEN |
| `TaskClaimed` | `workerId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `TaskStarted` | `startedAt` | CLAIMED → IN_PROGRESS |
| `ArtifactSubmitted` | `summary, sectionCount, submittedAt` | IN_PROGRESS → IN_REVIEW |
| `QualityPassed` | `qualityReport` | IN_REVIEW → DONE, `completedAt = now` |
| `QualityFailed` | `qualityReport` | IN_REVIEW → IN_PROGRESS (retry) |
| `TaskBlocked` | `reason, blockedAt` | (any active) → BLOCKED |
| `TaskReleased` | `releasedAt` | CLAIMED/IN_PROGRESS → OPEN, clears `claimedBy` |
| `TaskCompleted` | `completedAt` | (terminal marker emitted alongside `QualityPassed`; see note) |

Note: `QualityPassed` carries the `DONE` transition and `completedAt`; `TaskCompleted` is the explicit terminal event the goal-completion check subscribes to. Prefer two events for a clean audit trail.

## Events — `GoalEntity`

| Event | Payload | Transition |
|---|---|---|
| `GoalCreated` | `goalId, title, description, requestedBy, createdAt` | → PLANNING |
| `GoalPlanned` | `taskIds, planSummary` | PLANNING → READY |
| `GoalStarted` | `startedAt` | READY → IN_PROGRESS |
| `GoalCompleted` | `completedAt` | IN_PROGRESS → COMPLETED |

## Events — `WorkerMailbox`

| Event | Payload |
|---|---|
| `MessagePosted` | `PeerMessage` |
| `MessageReplied` | `messageId, reply, repliedAt` |

## Events — `IntakeQueue`

| Event | Payload |
|---|---|
| `GoalSubmitted` | `goalId, title, description, requestedBy, submittedAt` |

## Key-value state — `SystemControl`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `halted` | `boolean` | no | Whether the team is frozen. |
| `haltedReason` | `Optional<String>` | yes | Operator-supplied reason. |
| `haltedBy` | `Optional<String>` | yes | Operator id. |
| `haltedAt` | `Optional<Instant>` | yes | When halted. |

## View row

`TaskRow` mirrors `Task` but drops the heavy `OutputSection` contents — it keeps `artifactSummary` and `sectionCount` only, so the SSE stream stays small. Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6). The UI fetches nothing heavier; the section content itself is not projected into the view.
