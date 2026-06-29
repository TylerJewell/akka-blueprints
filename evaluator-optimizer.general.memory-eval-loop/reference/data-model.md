# Data model — memory-eval-loop

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Question` | `questionId` | `String` | no | Unique id (derived from sessionId at creation). |
| | `text` | `String` | no | The user's question. |
| | `userId` | `String` | no | Identifier of the submitting user. |
| `MemoryEntry` | `entryId` | `String` | no | Unique id for this entry. |
| | `userId` | `String` | no | Owner of this entry. |
| | `content` | `String` | no | The stored fact fragment (post-sanitization). |
| | `sourceSessionId` | `String` | no | Session that produced this entry. |
| | `recordedAt` | `Instant` | no | When the entry was written to the store. |
| `ScoringNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Score` | `verdict` | `ScorerVerdict` | no | `PASS` or `IMPROVE`. |
| | `notes` | `ScoringNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `scoredAt` | `Instant` | no | When the Scorer returned. |
| `AnswerAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `answerText` | `String` | no | The agent's answer prose. |
| | `citedEntryIds` | `List<String>` | no | Memory entry ids cited; may be empty if no relevant entries. |
| | `score` | `Optional<Score>` | yes | Empty at draft time; populated after the score step. |
| | `answeredAt` | `Instant` | no | When the AnswerAgent returned. |
| `Session` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `userId` | `String` | no | Submitting user. |
| | `questionText` | `String` | no | Original question text. |
| | `maxAttempts` | `int` | no | Per-session retry ceiling (default 3). |
| | `status` | `SessionStatus` | no | See enum. |
| | `attempts` | `List<AnswerAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `acceptedAttemptNumber` | `Optional<Integer>` | yes | Populated on `SessionAccepted`. |
| | `acceptedAnswer` | `Optional<String>` | yes | Populated on `SessionAccepted`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `SessionRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |

## Enums

`SessionStatus`: `ANSWERING`, `SCORING`, `ACCEPTED`, `REJECTED_FINAL`.

`ScorerVerdict`: `PASS`, `IMPROVE`.

## Events (`SessionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SessionCreated` | `sessionId, userId, questionText, maxAttempts, createdAt` | `SessionEndpoint POST /api/sessions` | → `ANSWERING` |
| `AnswerAttempted` | `attemptNumber, answerText, citedEntryIds, answeredAt` | After `answerStep` returns | (no status change; appends to `attempts[]`); on first attempt → `ANSWERING` already |
| `AttemptScored` | `attemptNumber, score: Score` | After `scoreStep` returns | Populates `attempts[n].score`; `PASS` → transition; `IMPROVE` and below max → `ANSWERING`; `IMPROVE` at max → `REJECTED_FINAL` |
| `SessionAccepted` | `attemptNumber, acceptedAnswer` | `Score.verdict = PASS` | → `ACCEPTED`, `finishedAt = now` |
| `SessionRejectedFinal` | `bestAttemptNumber, bestAnswer, rejectionReason` | `attempts.size() == maxAttempts` AND last score is `IMPROVE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, userId, recordedAt` | `EvalSampler` per scored attempt; workflow on terminal transition | (no status change; appended to an internal `evalEvents[]` for projection) |
| `DriftCheckRecorded` | `passRate, avgScore, driftFlagged, computedAt` | `DriftWatcher` every 10 min | (no status change; written to the sentinel entity `"drift-watch-singleton"`) |

## Events (`MemoryStore`)

| Event | Payload | Trigger |
|---|---|---|
| `MemoryEntryAdded` | `entryId, userId, content, sourceSessionId, recordedAt` | `AnswerWorkflow.memoryWriteStep` after session reaches terminal state |
| `MemoryEntryPiiRedacted` | `entryId, userId, redactedCount, types: List<String>, sourceSessionId` | `AnswerWorkflow.memoryWriteStep` when `PiiSanitizer` finds at least one match |

## View rows

### `SessionRow`

Structurally identical to `Session`. The `attempts` list is bounded at `maxAttempts` (default 3), keeping the row small enough for the SSE stream. The `SessionView`'s `TableUpdater` consumes every `SessionEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSessions` returning the full list. Callers filter by `status` client-side.

### `MemoryRow`

Structurally identical to `MemoryEntry`. The `MemoryView`'s `TableUpdater` appends one row per `MemoryEntryAdded` event; it does not delete rows. The view exposes one parameterized query: `getEntriesForUser(userId)`.
