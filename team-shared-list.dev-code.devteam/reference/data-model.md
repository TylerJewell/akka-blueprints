# Data model — dev-team-task-board

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ProjectBrief` | `projectId` | `String` | no | Id assigned at submission. |
| | `title` | `String` | no | Short project name. |
| | `description` | `String` | no | Free-text brief. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `TaskSpec` | `title` | `String` | no | Short imperative task title. |
| | `description` | `String` | no | One or two sentences a developer can act on. |
| | `dependsOn` | `List<String>` | no | Titles of tasks in the same plan that must finish first (may be empty). |
| `TaskPlan` | `planSummary` | `String` | no | One-sentence approach from the team lead. |
| | `tasks` | `List<TaskSpec>` | no | Three to six task specs. |
| `CodeFile` | `path` | `String` | no | Workspace-relative file path. |
| | `content` | `String` | no | File contents (no placeholder stubs). |
| `PeerRequest` | `toDeveloper` | `String` | no | Developer id the request is addressed to. |
| | `question` | `String` | no | Concrete coordination question. |
| `CodeArtifact` | `taskId` | `String` | no | The task this artifact completes. |
| | `files` | `List<CodeFile>` | no | One to three files (empty when a peer request is raised). |
| | `summary` | `String` | no | One-sentence description of the work. |
| | `peerRequest` | `Optional<PeerRequest>` | yes | Present only when the developer is blocked on a peer. |
| `TestReport` | `passed` | `boolean` | no | Whether the artifact passed the test gate. |
| | `failures` | `List<String>` | no | One string per failing file/check (empty on pass). |
| | `ranAt` | `Instant` | no | When the gate ran. |
| `PeerMessage` | `messageId` | `String` | no | Unique id. |
| | `fromDeveloper` | `String` | no | Sender developer id. |
| | `toDeveloper` | `String` | no | Recipient developer id. |
| | `taskId` | `String` | no | The blocked task the message concerns. |
| | `question` | `String` | no | The coordination question. |
| | `sentAt` | `Instant` | no | When posted. |
| | `reply` | `Optional<String>` | yes | Populated on reply. |
| | `repliedAt` | `Optional<Instant>` | yes | Populated on reply. |

## Entity state — `Task` (`TaskEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `taskId` | `String` | no | Deterministic id `projectId + "-t" + index`. |
| `projectId` | `String` | no | Owning project. |
| `title` | `String` | no | From the `TaskSpec`. |
| `description` | `String` | no | From the `TaskSpec`. |
| `dependsOn` | `List<String>` | no | Dependency titles. |
| `status` | `TaskStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Developer that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `artifactSummary` | `Optional<String>` | yes | Summary of the submitted artifact. |
| `fileCount` | `Optional<Integer>` | yes | Number of files in the artifact. |
| `testReport` | `Optional<TestReport>` | yes | Latest test-gate result. |
| `blockedReason` | `Optional<String>` | yes | Why the task is blocked. |
| `completedAt` | `Optional<Instant>` | yes | When the task reached `DONE`. |
| `createdAt` | `Instant` | no | When `TaskCreated` emitted. |

## Entity state — `Project` (`ProjectEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `projectId` | `String` | no | Unique id. |
| `title` | `String` | no | Project name. |
| `description` | `String` | no | Original brief. |
| `requestedBy` | `String` | no | Submitter id. |
| `status` | `ProjectStatus` | no | See enum. |
| `taskIds` | `List<String>` | no | Ids of tasks created from the plan. |
| `planSummary` | `Optional<String>` | yes | Populated on `ProjectPlanned`. |
| `createdAt` | `Instant` | no | When `ProjectCreated` emitted. |
| `completedAt` | `Optional<Instant>` | yes | When the project reached `COMPLETED`. |

## Enums

`TaskStatus`: `OPEN`, `CLAIMED`, `IN_PROGRESS`, `IN_REVIEW`, `DONE`, `BLOCKED`.
`ProjectStatus`: `PLANNING`, `READY`, `IN_PROGRESS`, `COMPLETED`.

## Events — `TaskEntity`

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `taskId, projectId, title, description, dependsOn, createdAt` | → OPEN |
| `TaskClaimed` | `developerId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `TaskStarted` | `startedAt` | CLAIMED → IN_PROGRESS |
| `ArtifactSubmitted` | `summary, fileCount, submittedAt` | IN_PROGRESS → IN_REVIEW |
| `TestPassed` | `testReport` | IN_REVIEW → DONE, `completedAt = now` |
| `TestFailed` | `testReport` | IN_REVIEW → IN_PROGRESS (retry) |
| `TaskBlocked` | `reason, blockedAt` | (any active) → BLOCKED |
| `TaskReleased` | `releasedAt` | CLAIMED/IN_PROGRESS → OPEN, clears `claimedBy` |
| `TaskCompleted` | `completedAt` | (terminal marker emitted with `TestPassed`; see note) |

Note: `TestPassed` carries the `DONE` transition and `completedAt`; `TaskCompleted` is the explicit terminal event the project-completion check subscribes to. An implementation may fold the two into one event if it keeps the projection and the completion check consistent — prefer two events for a clean audit trail.

## Events — `ProjectEntity`

| Event | Payload | Transition |
|---|---|---|
| `ProjectCreated` | `projectId, title, description, requestedBy, createdAt` | → PLANNING |
| `ProjectPlanned` | `taskIds, planSummary` | PLANNING → READY |
| `ProjectStarted` | `startedAt` | READY → IN_PROGRESS |
| `ProjectCompleted` | `completedAt` | IN_PROGRESS → COMPLETED |

## Events — `PeerMailbox`

| Event | Payload |
|---|---|
| `MessagePosted` | `PeerMessage` |
| `MessageReplied` | `messageId, reply, repliedAt` |

## Events — `IntakeQueue`

| Event | Payload |
|---|---|
| `ProjectSubmitted` | `projectId, title, description, requestedBy, submittedAt` |

## Key-value state — `SystemControl`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `halted` | `boolean` | no | Whether the team is frozen. |
| `haltedReason` | `Optional<String>` | yes | Operator-supplied reason. |
| `haltedBy` | `Optional<String>` | yes | Operator id. |
| `haltedAt` | `Optional<Instant>` | yes | When halted. |

## View row

`TaskRow` mirrors `Task` but drops the heavy `CodeFile` contents — it keeps `artifactSummary` and `fileCount` only, so the SSE stream stays small. Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6). The UI fetches nothing heavier; the artifact files themselves are not projected (this sample stores only the summary on the entity).
