# Data model — project-tracker

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RawBoardEvent` | `eventId` | `String` | no | Unique id from the board source. |
| | `eventType` | `String` | no | `"task.created"` or `"task.updated"`. |
| | `taskId` | `String` | no | The board's task identifier. |
| | `title` | `String` | no | Task title. |
| | `description` | `String` | no | Full task description. |
| | `assigneeId` | `Optional<String>` | yes | Pre-assigned owner, if any. |
| | `assigneeName` | `Optional<String>` | yes | Human-readable name of pre-assigned owner. |
| | `dueDate` | `Optional<Instant>` | yes | Task deadline. |
| | `occurredAt` | `Instant` | no | When the board emitted the event. |
| `TaskSummary` | `taskId` | `String` | no | — |
| | `title` | `String` | no | — |
| | `description` | `String` | no | — |
| | `dueDate` | `Optional<Instant>` | yes | — |
| | `priority` | `TaskPriority` | no | One of `URGENT`, `HIGH`, `NORMAL`, `LOW`. |
| `AssignmentRecommendation` | `recommendedOwnerId` | `String` | no | Team member identifier. |
| | `recommendedOwnerName` | `String` | no | Human-readable name. |
| | `rationale` | `String` | no | One short sentence. |
| | `confidence` | `String` | no | `"high" \| "medium" \| "low"`. |
| `NudgeDraft` | `taskId` | `String` | no | — |
| | `ownerId` | `String` | no | Target of the nudge. |
| | `message` | `String` | no | 2–4 sentence plain-text message. |
| | `tone` | `NudgeTone` | no | One of `FRIENDLY`, `DIRECT`, `ESCALATORY`. |
| | `draftedAt` | `Instant` | no | When the agent finished. |
| `DispatchResult` | `actionId` | `String` | no | Unique identifier for this guardrail check. |
| | `toolName` | `String` | no | `"assignTask"`, `"sendNudge"`, or `"escalateTask"`. |
| | `dispatched` | `boolean` | no | True = tool was called; false = blocked. |
| | `blockReason` | `Optional<String>` | yes | Populated when `dispatched = false`. |
| | `attemptedAt` | `Instant` | no | When the check ran. |
| `EvalResult` | `score` | `Integer` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the score driver. |
| `PlannerTask` (entity state) | `taskId` | `String` | no | — |
| | `summary` | `TaskSummary` | no | Captured once at create. |
| | `assignment` | `Optional<AssignmentRecommendation>` | yes | Populated after TaskAssigned. |
| | `currentOwnerId` | `Optional<String>` | yes | Set on TaskAssigned; cleared on reassignment. |
| | `currentOwnerName` | `Optional<String>` | yes | Human-readable form of currentOwnerId. |
| | `nudgeCount` | `int` | no | Increments on each NudgeSent. |
| | `lastNudgedAt` | `Optional<Instant>` | yes | Timestamp of the most recent NudgeSent. |
| | `evalScore` | `Optional<Integer>` | yes | Populated after AssignmentEvalScored. |
| | `evalRationale` | `Optional<String>` | yes | Populated after AssignmentEvalScored. |
| | `status` | `TaskStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When TaskCreated emitted. |
| | `completedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

## Enums

`TaskPriority`: `URGENT`, `HIGH`, `NORMAL`, `LOW`.
`NudgeTone`: `FRIENDLY`, `DIRECT`, `ESCALATORY`.
`TaskStatus`: `CREATED`, `ASSIGNED`, `IN_PROGRESS`, `OVERDUE`, `NUDGED`, `ESCALATED`, `COMPLETED`, `CANCELLED`.

## Events (`PlannerTaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `summary, createdAt` | → CREATED |
| `TaskAssigned` | `assignment, currentOwnerId, currentOwnerName` | → ASSIGNED |
| `TaskAssignmentBlocked` | `dispatchResult` | stays CREATED |
| `TaskProgressStarted` | — | → IN_PROGRESS |
| `TaskOverdue` | — | → OVERDUE (from ASSIGNED or IN_PROGRESS) |
| `NudgeSent` | `nudgeDraft, dispatchResult, sentAt` | → NUDGED; nudgeCount++ |
| `TaskEscalated` | `escalatedBy, reason, escalatedAt` | → ESCALATED |
| `TaskCompleted` | `resolvedBy, completedAt` | → COMPLETED (terminal) |
| `TaskCancelled` | `cancelledBy, reason` | → CANCELLED (terminal) |
| `AssignmentEvalScored` | `score, rationale` | stays COMPLETED; populates evalScore + evalRationale |

## Events (`BoardEventQueue`)

| Event | Payload |
|---|---|
| `BoardEventReceived` | `rawBoardEvent` (the pre-normalisation payload — used by the audit log) |

## View row

`BoardTaskRow` mirrors `PlannerTask`. The UI fetches full detail via `GET /api/board/{id}`; the view row is used for the list. All Optional fields on the view row are `Optional<T>` — no null.
