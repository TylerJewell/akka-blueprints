# Data model — self-correcting-extraction

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DocumentSubmission` | `documentType` | `String` | no | Declared type (e.g., `invoice`, `purchase-order`). |
| | `rawText` | `String` | no | Full text of the submitted document. |
| | `submittedBy` | `String` | no | UI identifier of the submitter; defaults to `"anonymous"`. |
| `FieldMap` | `fields` | `Map<String, String>` | no | Extracted field name → value pairs. |
| | `confidence` | `double` | no | Agent's self-reported confidence in the extraction (`0.0–1.0`). |
| | `extractedAt` | `Instant` | no | When the ExtractionAgent returned this map. |
| `CorrectionNotes` | `bullets` | `List<String>` | no | Up to 4 short bullets naming the field, the found value, and the expected form. |
| | `overallRationale` | `String` | no | One-sentence summary; required whether decision is PASS or CORRECT. |
| `ScorerVerdict` | `decision` | `ScorerDecision` | no | `PASS` or `CORRECT`. |
| | `notes` | `CorrectionNotes` | no | See above. |
| | `confidence` | `double` | no | Scorer's confidence in the extraction as-is (`0.0–1.0`). |
| | `scoredAt` | `Instant` | no | When the ScorerAgent returned this verdict. |
| `ExtractionAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `fieldMap` | `FieldMap` | no | The ExtractionAgent's output for this attempt. |
| | `verdict` | `Optional<ScorerVerdict>` | yes | Empty if the workflow was halted before scoring this attempt. |
| `ExtractionJob` (entity state) | `jobId` | `String` | no | Unique id. |
| | `documentType` | `String` | no | Declared document type. |
| | `rawText` | `String` | no | Original submitted text. |
| | `budgetCap` | `int` | no | Per-job attempt ceiling (default 4). |
| | `status` | `JobStatus` | no | See enum. |
| | `attempts` | `List<ExtractionAttempt>` | no | Bounded at `budgetCap`; starts empty. |
| | `verifiedAttemptNumber` | `Optional<Integer>` | yes | Populated on `JobVerified`. |
| | `verifiedFieldMap` | `Optional<FieldMap>` | yes | Populated on `JobVerified`. |
| | `exhaustionReason` | `Optional<String>` | yes | Populated on `JobBudgetExhausted`. |
| | `createdAt` | `Instant` | no | When `JobCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |
| `MemoryRecord` | `documentType` | `String` | no | Document type this record applies to. |
| | `fieldName` | `String` | no | The confirmed field name. |
| | `confirmedValue` | `String` | no | The confirmed value string. |
| | `confirmedAt` | `Instant` | no | When this field value was last confirmed. |

## Enums

`JobStatus`: `EXTRACTING`, `SCORING`, `VERIFIED`, `BUDGET_EXHAUSTED`.

`ScorerDecision`: `PASS`, `CORRECT`.

## Events (`ExtractionJobEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `JobCreated` | `jobId, documentType, rawText, budgetCap, createdAt` | Workflow `startStep` | → `EXTRACTING` |
| `AttemptExtracted` | `attemptNumber, fieldMap: FieldMap` | After `extractStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptScored` | `attemptNumber, verdict: ScorerVerdict` | After `scoreStep` returns | `PASS` → `SCORING` (transient, then `JobVerified`); `CORRECT` → `EXTRACTING` if attempts < cap, else `BUDGET_EXHAUSTED` |
| `JobVerified` | `attemptNumber, verifiedFieldMap: FieldMap` | `ScorerDecision = PASS` | → `VERIFIED`, `finishedAt = now` |
| `JobBudgetExhausted` | `bestAttemptNumber, bestFieldMap: FieldMap, exhaustionReason` | `attempts.size() == budgetCap` AND last verdict is `CORRECT`; OR `defaultStepRecovery` failover | → `BUDGET_EXHAUSTED`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, decision, confidence, memoryHit, recordedAt` | `EvalSampler` per scored attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`MemoryEntity`)

| Event | Payload | Trigger |
|---|---|---|
| `MemoryRecordWritten` | `documentType, fieldName, confirmedValue, confirmedAt` | `verifyStep` in `ExtractionWorkflow` after `JobVerified` |

## Events (`DocumentQueue`)

| Event | Payload |
|---|---|
| `DocumentSubmitted` | `jobId, documentType, rawText, submittedBy, submittedAt` |

## View row

`JobRow` is structurally identical to `ExtractionJob` — the `attempts` list is bounded at `budgetCap` (default 4) so the row stays small enough to push down the SSE stream. The view's `TableUpdater` consumes every `ExtractionJobEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllJobs` returning the full list. Callers filter by `status` client-side.
