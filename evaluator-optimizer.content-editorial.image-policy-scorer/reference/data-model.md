# Data model — image-policy-scorer

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ImagePrompt` | `promptText` | `String` | no | The user-submitted image brief. |
| | `audienceTier` | `String` | no | One of `general`, `mature`, `children`. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ImageDescription` | `descriptionText` | `String` | no | The visual description produced by the generator. |
| | `contentCategory` | `String` | no | One of `product`, `lifestyle`, `nature`, `portrait`, `abstract`, `editorial`. |
| | `brandSafetySignal` | `String` | no | One of `none`, `mild`, `violence`, `adult-explicit`, `hate-speech`, `self-harm`. |
| | `generatedAt` | `Instant` | no | When the GeneratorAgent returned the description. |
| `SafetyGateVerdict` | `passed` | `boolean` | no | Whether the gate accepted the description. |
| | `reasonCode` | `String` | no | `OK`, `PROHIBITED_SIGNAL`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `PolicyNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `PolicyVerdict` | `decision` | `ScorerDecision` | no | `PASS` or `FAIL`. |
| | `notes` | `PolicyNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric minimum across four dimensions. |
| | `scoredAt` | `Instant` | no | When the ScorerAgent returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes gate-blocked attempts). |
| | `description` | `ImageDescription` | no | The GeneratorAgent's output for this attempt. |
| | `gateVerdict` | `SafetyGateVerdict` | no | The deterministic check's verdict. |
| | `policyVerdict` | `Optional<PolicyVerdict>` | yes | Empty when the gate blocked; populated otherwise. |
| `Image` (entity state) | `imageId` | `String` | no | Unique id. |
| | `promptText` | `String` | no | Original brief. |
| | `audienceTier` | `String` | no | Audience tier label. |
| | `maxAttempts` | `int` | no | Per-image retry ceiling (default 4). |
| | `status` | `ImageStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `ImageApproved`. |
| | `approvedDescription` | `Optional<String>` | yes | Populated on `ImageApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `ImageRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `ImageCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the image reached a terminal state. |

## Enums

`ImageStatus`: `GENERATING`, `SCORING`, `APPROVED`, `REJECTED_FINAL`.

`ScorerDecision`: `PASS`, `FAIL`.

## Events (`ImageEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ImageCreated` | `promptText, audienceTier, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, description: ImageDescription` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGateVerdictRecorded` | `attemptNumber, verdict: SafetyGateVerdict` | After `gateStep` | `passed=true` → `SCORING`; `passed=false` → `GENERATING` (re-generate) |
| `AttemptScored` | `attemptNumber, verdict: PolicyVerdict` | After `scoreStep` returns | (no status change; populates `attempts[n].policyVerdict`) |
| `ImageApproved` | `attemptNumber, approvedDescription` | `PolicyVerdict.decision = PASS` | → `APPROVED`, `finishedAt = now` |
| `ImageRejectedFinal` | `bestAttemptNumber, bestDescription, rejectionReason` | `attempts.size() == maxAttempts` AND last verdict is `FAIL`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `PolicyEvalRecorded` | `attemptNumber, decision, score, gateBlocked, recordedAt` | `EvalSampler` per scored attempt; workflow on terminal transition | (no status change; appended to an internal `evalEvents[]` view-side projection) |

## Events (`PromptQueue`)

| Event | Payload |
|---|---|
| `PromptSubmitted` | `imageId, promptText, audienceTier, requestedBy, submittedAt` |

## View row

`ImageRow` is structurally identical to `Image` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `ImageEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllImages` returning the full list. Callers filter by `status` client-side.
