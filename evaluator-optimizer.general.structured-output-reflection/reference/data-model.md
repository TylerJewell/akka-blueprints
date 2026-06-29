# Data model — structured-output-reflection

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GenerationRequest` | `schemaName` | `String` | no | The target schema (`product`, `event-record`, `contact`). |
| | `prompt` | `String` | no | Natural-language content description. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `GeneratedOutput` | `json` | `String` | no | The generated JSON document as a raw string. |
| | `tokenCount` | `int` | no | Estimated token count of `json`. |
| | `parseable` | `boolean` | no | Whether the output is syntactically valid JSON. |
| | `generatedAt` | `Instant` | no | When the Generator returned the output. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the output. |
| | `reasonCode` | `String` | no | `OK`, `SCHEMA_INVALID`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ValidationNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `ValidationReport` | `verdict` | `CriticVerdict` | no | `PASS` or `REVISE`. |
| | `notes` | `ValidationNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the Critic returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `output` | `GeneratedOutput` | no | The Generator's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `report` | `Optional<ValidationReport>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Generation` (entity state) | `generationId` | `String` | no | Unique id. |
| | `schemaName` | `String` | no | Target schema. |
| | `prompt` | `String` | no | Original content description. |
| | `maxAttempts` | `int` | no | Per-generation retry ceiling (default 4). |
| | `status` | `GenerationStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `passedAttemptNumber` | `Optional<Integer>` | yes | Populated on `GenerationPassed`. |
| | `passedJson` | `Optional<String>` | yes | Populated on `GenerationPassed`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `GenerationFailedFinal`. |
| | `createdAt` | `Instant` | no | When `GenerationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the generation reached a terminal state. |

## Enums

`GenerationStatus`: `GENERATING`, `VALIDATING`, `PASSED`, `FAILED_FINAL`.

`CriticVerdict`: `PASS`, `REVISE`.

## Events (`GenerationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `GenerationCreated` | `schemaName, prompt, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, output: GeneratedOutput` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `VALIDATING`; `passed=false` → `GENERATING` (re-generate) |
| `AttemptValidated` | `attemptNumber, report: ValidationReport` | After `validateStep` returns | (no status change; populates `attempts[n].report`) |
| `GenerationPassed` | `attemptNumber, passedJson` | `ValidationReport.verdict = PASS` | → `PASSED`, `finishedAt = now` |
| `GenerationFailedFinal` | `bestAttemptNumber, bestJson, failureReason` | `attempts.size() == maxAttempts` AND last report is `REVISE`; OR `defaultStepRecovery` failover | → `FAILED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, schemaFailed, recordedAt` | `EvalSampler` per validated attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `RequestSubmitted` | `generationId, schemaName, prompt, requestedBy, submittedAt` |

## View row

`GenerationRow` is structurally identical to `Generation` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `GenerationEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllGenerations` returning the full list. Callers filter by `status` client-side.
