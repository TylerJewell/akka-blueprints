# Data model — self-improving-deep-researcher

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ResearchQuery` | `topic` | `String` | no | The user-submitted research topic. |
| | `depthHint` | `String` | no | `brief`, `standard`, or `deep`. |
| | `sourceTypePreference` | `String` | no | `any`, `primary`, `secondary`, or `peer-reviewed`. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `EvidenceSegment` | `claim` | `String` | no | One declarative sentence supporting the report. |
| | `sourceRef` | `String` | no | URL or citation for the claim. |
| | `confidence` | `double` | no | 0.0–1.0; how well-supported the claim is. |
| `SourceManifest` | `sourceRefs` | `List<String>` | no | Deduplicated list of all source references used. |
| | `totalSources` | `int` | no | Count of distinct sources. |
| `ResearchReport` | `executiveSummary` | `String` | no | 2–4 sentence analytical prose synthesis. |
| | `evidence` | `List<EvidenceSegment>` | no | All evidence segments for this attempt. |
| | `sources` | `SourceManifest` | no | Deduplicated source list for this attempt. |
| | `researchedAt` | `Instant` | no | When the ResearchAgent returned the report. |
| `QualityNotes` | `bullets` | `List<String>` | no | 0–4 short critique bullets (empty on `SUFFICIENT`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `QualityEvaluation` | `verdict` | `EvaluatorVerdict` | no | `SUFFICIENT` or `REFINE`. |
| | `notes` | `QualityNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric minimum across four dimensions. |
| | `evaluatedAt` | `Instant` | no | When the EvaluatorAgent returned. |
| `MemoryBlock` | `blockId` | `String` | no | Identifier: `role`, `strategy`, `source-prefs`, `synthesis-style`. |
| | `blockType` | `String` | no | One of the four block type names. |
| | `content` | `String` | no | Current block text. |
| | `updatedAt` | `Instant` | no | When this block was last changed. |
| `PromptMemory` | `blocks` | `List<MemoryBlock>` | no | All current memory blocks (4 at baseline). |
| | `fingerprint` | `String` | no | First 16 hex chars of SHA-256 of serialized `blocks`. |
| | `lastModifiedAt` | `Instant` | no | When the last block update was written. |
| `MemoryBlockChange` | `blockId` | `String` | no | Which block this change targets. |
| | `changeType` | `String` | no | `UPDATE`, `APPEND`, or `NO_CHANGE`. |
| | `before` | `String` | no | Content before the change (verbatim copy). |
| | `after` | `String` | no | Content after the change; empty string for `NO_CHANGE`. |
| | `reason` | `String` | no | One sentence explaining the change. |
| `MemoryDiff` | `changes` | `List<MemoryBlockChange>` | no | All reviewed blocks (including `NO_CHANGE` entries). |
| | `triggerSessionId` | `String` | no | The session that caused this diff. |
| | `triggerVerdict` | `EvaluatorVerdict` | no | The terminal verdict that triggered the rewrite. |
| | `diffedAt` | `Instant` | no | When the PromptRewriterAgent returned. |
| `ResearchAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `report` | `ResearchReport` | no | The ResearchAgent's output for this attempt. |
| | `evaluation` | `Optional<QualityEvaluation>` | yes | Empty until the EvaluatorAgent returns. |
| `Session` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `topic` | `String` | no | Original research topic. |
| | `depthHint` | `String` | no | Requested depth. |
| | `maxAttempts` | `int` | no | Per-session retry ceiling (default 3). |
| | `status` | `SessionStatus` | no | See enum. |
| | `attempts` | `List<ResearchAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `acceptedAttemptNumber` | `Optional<Integer>` | yes | Populated on `SessionAccepted`. |
| | `acceptedReport` | `Optional<ResearchReport>` | yes | Populated on `SessionAccepted`. |
| | `memoryDiff` | `Optional<MemoryDiff>` | yes | Populated on `MemoryUpdated`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal + memory-rewrite state. |

## Enums

`SessionStatus`: `RESEARCHING`, `EVALUATING`, `ACCEPTED`, `MAX_ATTEMPTS_REACHED`.

`EvaluatorVerdict`: `SUFFICIENT`, `REFINE`.

## Events (`SessionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SessionCreated` | `topic, depthHint, maxAttempts, createdAt` | Workflow `startStep` | → `RESEARCHING` |
| `AttemptResearched` | `attemptNumber, report: ResearchReport` | After `researchStep` or `refineStep` returns | `RESEARCHING` (appends to `attempts[]`); → `EVALUATING` when evaluator step begins |
| `AttemptEvaluated` | `attemptNumber, evaluation: QualityEvaluation` | After `evaluateStep` returns | `EVALUATING`; populates `attempts[n].evaluation`; no status change until routing decision |
| `SessionAccepted` | `attemptNumber, acceptedReport` | `QualityEvaluation.verdict = SUFFICIENT` | → `ACCEPTED`; `acceptedAttemptNumber` + `acceptedReport` populated |
| `SessionMaxAttemptsReached` | `bestAttemptNumber, bestReport, reason` | `attempts.size() == maxAttempts` AND last verdict is `REFINE`; OR `defaultStepRecovery` failover | → `MAX_ATTEMPTS_REACHED` |
| `MemoryUpdated` | `diff: MemoryDiff` | After `rewriteMemoryStep` completes (always, regardless of terminal state) | `finishedAt = now`; `memoryDiff` populated |
| `QualityEvalRecorded` | `attemptNumber, verdict, score, recordedAt` | `DriftSampler` per evaluated attempt; workflow on `MemoryUpdated` | (no status change; appended to an internal `qualityEvalEvents[]` view-side projection) |
| `PromptDriftRecorded` | `oldFingerprint, newFingerprint, changedBlockCount, detectedAt` | `DriftSampler` on fingerprint mismatch | (no status change; surfaced in the drift timeline) |

## Events (`QueryQueue`)

| Event | Payload |
|---|---|
| `QuerySubmitted` | `sessionId, topic, depthHint, sourceTypePreference, requestedBy, submittedAt` |

## View row

`SessionRow` is structurally identical to `Session` — the `attempts` list is bounded at `maxAttempts` (default 3) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SessionEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSessions` returning the full list. Callers filter by `status` client-side.
