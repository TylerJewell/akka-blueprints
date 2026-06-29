# Data model — sql-gen-validator-loop

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `NlQuestion` | `text` | `String` | no | The user-submitted natural-language question. |
| | `schemaName` | `String` | no | Target schema name for the generated SQL. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `GeneratedQuery` | `sql` | `String` | no | The SQL statement (SELECT only). |
| | `dialect` | `String` | no | SQL dialect; default `"standard-sql"`. |
| | `generatedAt` | `Instant` | no | When the GeneratorAgent returned the query. |
| `MutationGuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the query. |
| | `reasonCode` | `String` | no | `OK`, `MUTATION_FORBIDDEN`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ValidationNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `VALID`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `ValidationResult` | `verdict` | `ValidatorVerdict` | no | `VALID` or `INVALID`. |
| | `notes` | `ValidationNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the ValidatorAgent returned. |
| `QueryAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes mutation-blocked attempts). |
| | `query` | `GeneratedQuery` | no | The GeneratorAgent's output for this attempt. |
| | `guardrail` | `MutationGuardrailVerdict` | no | The deterministic check's verdict. |
| | `validation` | `Optional<ValidationResult>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `QueryRequest` (entity state) | `requestId` | `String` | no | Unique id. |
| | `question` | `String` | no | Original natural-language question. |
| | `schemaName` | `String` | no | Target schema name. |
| | `maxAttempts` | `int` | no | Per-request retry ceiling (default 4). |
| | `status` | `QueryStatus` | no | See enum. |
| | `attempts` | `List<QueryAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `acceptedAttemptNumber` | `Optional<Integer>` | yes | Populated on `QueryRequestAccepted`. |
| | `acceptedSql` | `Optional<String>` | yes | Populated on `QueryRequestAccepted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `QueryRequestFailedFinal`. |
| | `createdAt` | `Instant` | no | When `QueryRequestCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the request reached a terminal state. |

## Enums

`QueryStatus`: `GENERATING`, `VALIDATING`, `ACCEPTED`, `FAILED_FINAL`.

`ValidatorVerdict`: `VALID`, `INVALID`.

## Events (`QueryRequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QueryRequestCreated` | `question, schemaName, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, query: GeneratedQuery` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptMutationGuardrailVerdictRecorded` | `attemptNumber, verdict: MutationGuardrailVerdict` | After `guardrailStep` | `passed=true` → `VALIDATING`; `passed=false` → `GENERATING` (re-generate) |
| `AttemptValidated` | `attemptNumber, validation: ValidationResult` | After `validateStep` returns | (no status change; populates `attempts[n].validation`) |
| `QueryRequestAccepted` | `attemptNumber, acceptedSql` | `ValidationResult.verdict = VALID` | → `ACCEPTED`, `finishedAt = now` |
| `QueryRequestFailedFinal` | `bestAttemptNumber, bestSql, failureReason` | `attempts.size() == maxAttempts` AND last validation is `INVALID`; OR `defaultStepRecovery` failover | → `FAILED_FINAL`, `finishedAt = now` |
| `ValidationEvalRecorded` | `attemptNumber, verdict, score, mutationBlocked, recordedAt` | `EvalSampler` per validated attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `QuestionSubmitted` | `requestId, text, schemaName, requestedBy, submittedAt` |

## View row

`QueryRequestRow` is structurally identical to `QueryRequest` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `QueryRequestEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllRequests` returning the full list. Callers filter by `status` client-side.
