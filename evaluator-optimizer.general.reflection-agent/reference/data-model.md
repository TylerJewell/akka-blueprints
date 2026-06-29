# Data model — reflection-agent

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskPrompt` | `text` | `String` | no | The user-submitted task prompt. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `GeneratedResponse` | `text` | `String` | no | The generated response. |
| | `wordCount` | `int` | no | Word count of `text`. |
| | `generatedAt` | `Instant` | no | When the Generator returned the response. |
| `ReflectionNotes` | `changeRequests` | `List<String>` | no | 0–3 specific change requests (empty on `ACCEPT`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Reflection` | `verdict` | `ReflectorVerdict` | no | `ACCEPT` or `REVISE`. |
| | `notes` | `ReflectionNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric (minimum of four dimensions). |
| | `reflectedAt` | `Instant` | no | When the Reflector returned. |
| `Iteration` | `iterationNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `response` | `GeneratedResponse` | no | The Generator's output for this iteration. |
| | `reflection` | `Optional<Reflection>` | yes | Empty while the iteration is still in `GENERATING`; populated after the Reflector runs. |
| `Task` (entity state) | `taskId` | `String` | no | Unique id. |
| | `promptText` | `String` | no | Original task prompt. |
| | `maxIterations` | `int` | no | Per-task iteration ceiling (default 4). |
| | `status` | `TaskStatus` | no | See enum. |
| | `iterations` | `List<Iteration>` | no | Bounded at `maxIterations`; starts empty. |
| | `acceptedIterationNumber` | `Optional<Integer>` | yes | Populated on `TaskAccepted`. |
| | `acceptedResponse` | `Optional<String>` | yes | Populated on `TaskAccepted`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `TaskRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `TaskCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the task reached a terminal state. |

## Enums

`TaskStatus`: `GENERATING`, `REFLECTING`, `ACCEPTED`, `REJECTED_FINAL`.

`ReflectorVerdict`: `ACCEPT`, `REVISE`.

## Events (`TaskEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TaskCreated` | `taskId, promptText, maxIterations, createdAt` | Workflow `startStep` | → `GENERATING` |
| `IterationGenerated` | `iterationNumber, response: GeneratedResponse` | After `generateStep` returns | → `REFLECTING` (appends to `iterations[]`) |
| `IterationReflected` | `iterationNumber, reflection: Reflection` | After `reflectStep` returns | (no status change; populates `iterations[n].reflection`) |
| `TaskAccepted` | `iterationNumber, acceptedResponse` | `Reflection.verdict = ACCEPT` | → `ACCEPTED`, `finishedAt = now` |
| `TaskRejectedFinal` | `bestIterationNumber, bestResponse, rejectionReason` | `iterations.size() == maxIterations` AND last reflection is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `iterationNumber, verdict, score, recordedAt` | `EvalSampler` per reflected iteration; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `taskId, text, submittedBy, submittedAt` |

## View row

`TaskRow` is structurally identical to `Task` — the `iterations` list is bounded at `maxIterations` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `TaskEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllTasks` returning the full list. Callers filter by `status` client-side.
