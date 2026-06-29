# Data model — task-insight-memory

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskSpec` | `taskType` | `String` | no | Short identifier for the task category. |
| | `description` | `String` | no | Full free-text description of the task. |
| | `acceptanceCriteria` | `String` | no | Standard the result must meet; defaults to "answer is accurate and complete". |
| | `requestedBy` | `String` | no | Submitter identifier; defaults to "anonymous". |
| `RetrievedInsight` | `insightId` | `String` | no | ID of the insight pulled from the memory store. |
| | `taskType` | `String` | no | Task type the insight belongs to. |
| | `text` | `String` | no | Sanitized insight text. |
| | `confidence` | `double` | no | Confidence score at time of original persistence. |
| | `provenance` | `InsightProvenance` | no | `CORRECTION`, `DEMONSTRATION`, or `VERIFIED_EXPERIENCE`. |
| `TaskResult` | `answer` | `String` | no | The executor's primary answer. |
| | `confidence` | `double` | no | Self-assessed confidence in [0.0, 1.0]. |
| | `keyFindings` | `List<String>` | no | 2–5 reusable observations for future executions of the same task type. |
| | `completedAt` | `Instant` | no | When the executor returned. |
| `EvalNotes` | `bullets` | `List<String>` | no | 0–3 specific shortfall bullets (empty on `VERIFIED`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `EvalVerdict` | `outcome` | `EvalOutcome` | no | `VERIFIED` or `REJECTED`. |
| | `notes` | `EvalNotes` | no | See above. |
| | `qualityScore` | `double` | no | Minimum of the three rubric dimensions; in [0.0, 1.0]. |
| | `evaluatedAt` | `Instant` | no | When the evaluator returned. |
| `InsightCandidate` | `taskType` | `String` | no | Task type the insight belongs to. |
| | `sanitizedText` | `String` | no | PII-scrubbed joined key findings. |
| | `confidence` | `double` | no | Confidence from the originating `TaskResult`. |
| | `provenance` | `InsightProvenance` | no | Always `VERIFIED_EXPERIENCE` for workflow-generated insights. |
| `MemoryInsight` | `insightId` | `String` | no | Unique id. |
| | `taskType` | `String` | no | Task type. |
| | `text` | `String` | no | Sanitized insight text. |
| | `confidence` | `double` | no | Confidence score. |
| | `provenance` | `InsightProvenance` | no | Write-path identifier. |
| | `persistedAt` | `Instant` | no | When the insight was first written. |
| | `supersededAt` | `Optional<Instant>` | yes | Populated when a correction supersedes this insight. |
| `TaskRecord` (entity state) | `taskId` | `String` | no | Unique id. |
| | `taskType` | `String` | no | Task category. |
| | `description` | `String` | no | Original task description. |
| | `acceptanceCriteria` | `String` | no | Acceptance standard. |
| | `status` | `TaskStatus` | no | See enum. |
| | `retrievedInsights` | `List<RetrievedInsight>` | no | Insights retrieved at the start of the workflow; starts empty. |
| | `result` | `Optional<TaskResult>` | yes | Populated after `TaskResultRecorded`. |
| | `evalVerdict` | `Optional<EvalVerdict>` | yes | Populated after `TaskEvalVerdictRecorded`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `TaskFailed`. |
| | `createdAt` | `Instant` | no | When `TaskCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the task reached a terminal state. |
| `SnapshotMetrics` | `typeDistribution` | `Map<String, Integer>` | no | Count of active insights per task type. |
| | `avgConfidence` | `double` | no | Moving average confidence across all non-superseded insights. |
| | `snapshotAt` | `Instant` | no | When the snapshot was taken. |
| `DriftReason` | `concentrationRatio` | `double` | no | Fraction of insights belonging to the dominant task type. |
| | `avgConfidence` | `double` | no | Current moving-average confidence. |
| | `dominantTaskType` | `String` | no | Task type with the highest insight count. |
| | `triggeredThreshold` | `String` | no | `TYPE_CONCENTRATION`, `CONFIDENCE_FLOOR`, or `BOTH`. |

## Enums

`TaskStatus`: `PENDING`, `EXECUTING`, `EVALUATED`, `VERIFIED`, `FAILED`.

`EvalOutcome`: `VERIFIED`, `REJECTED`.

`InsightProvenance`: `CORRECTION`, `DEMONSTRATION`, `VERIFIED_EXPERIENCE`.

## Events (`TaskEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TaskCreated` | `taskId, taskType, description, acceptanceCriteria, createdAt` | Workflow `startStep` | → `PENDING` |
| `TaskExecutionStarted` | `taskId, retrievedInsights` | After `retrieveStep` completes | → `EXECUTING` |
| `TaskResultRecorded` | `taskId, result: TaskResult` | After `executeStep` returns | (no status change; populates `result`) |
| `TaskEvalVerdictRecorded` | `taskId, evalVerdict: EvalVerdict` | After `evaluateStep` returns | → `EVALUATED` |
| `TaskVerified` | `taskId, insightId, finishedAt` | After `persistStep` emits `InsightPersisted` | → `VERIFIED` |
| `TaskFailed` | `taskId, failureReason, finishedAt` | `outcome=REJECTED` OR `confidence < threshold` OR `defaultStepRecovery` failover | → `FAILED` |

## Events (`MemoryEntity`)

| Event | Payload | Trigger |
|---|---|---|
| `InsightPersisted` | `insightId, taskType, sanitizedText, confidence, provenance=VERIFIED_EXPERIENCE, persistedAt` | Workflow `persistStep` |
| `InsightSuperseded` | `insightId, supersededAt, replacedBy` | `POST /api/memory/correction` with `supersedes` field |
| `CorrectionApplied` | `insightId, taskType, correctedText, confidence, persistedAt` | `POST /api/memory/correction` |
| `DemonstrationAdded` | `insightId, taskType, demonstrationText, persistedAt` | `POST /api/memory/demonstration` |
| `DriftAssessmentRecorded` | `assessmentId, reason: DriftReason, assessedAt` | `DriftWatcher` when threshold crossed |

## Events (`TaskQueue`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `taskId, taskType, description, acceptanceCriteria, requestedBy, submittedAt` |

## View rows

**`InsightRow`** (in `MemoryView`) mirrors `MemoryInsight`. Two queries:
- `getAllInsights` — full store, no filter.
- `getInsightsByTaskType(taskType)` — filtered by task type, ordered by `persistedAt DESC`, excluding rows where `supersededAt` is non-null.

**`TaskRow`** (in `TaskView`) mirrors `TaskRecord`. One query:
- `getAllTasks` — full list. Callers filter by `status` client-side because Akka cannot auto-index enum columns (Lesson 2).
