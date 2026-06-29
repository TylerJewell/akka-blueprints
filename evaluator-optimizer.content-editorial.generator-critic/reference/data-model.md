# Data model — generator-critic

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TopicRequest` | `topic` | `String` | no | The user-submitted writing prompt. |
| | `wordCeiling` | `int` | no | Per-essay hard cap on draft word count. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `EssayDraft` | `text` | `String` | no | The essay body itself. |
| | `wordCount` | `int` | no | Word count of `text`. |
| | `draftedAt` | `Instant` | no | When the Generator returned the draft. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the content-policy guardrail accepted the draft. |
| | `reasonCode` | `String` | no | `OK` or `POLICY_VIOLATION`. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ReflectionNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `ACCEPT`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Reflection` | `verdict` | `ReflectorVerdict` | no | `ACCEPT` or `REVISE`. |
| | `notes` | `ReflectionNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the Reflector returned. |
| `Round` | `roundNumber` | `int` | no | 1-indexed; monotonic across the reflect-revise loop. |
| | `draft` | `EssayDraft` | no | The Generator's output for this round. |
| | `guardrail` | `GuardrailVerdict` | no | Populated when the guardrail ran for this round; `OK` with empty detail when not triggered during the inner loop. |
| | `reflection` | `Optional<Reflection>` | yes | Empty during `DRAFTING`; populated after the Reflector returns. |
| `Essay` (entity state) | `essayId` | `String` | no | Unique id. |
| | `topic` | `String` | no | Original writing prompt. |
| | `wordCeiling` | `int` | no | Per-essay cap. |
| | `maxRounds` | `int` | no | Per-essay retry ceiling (default 4). |
| | `status` | `EssayStatus` | no | See enum. |
| | `rounds` | `List<Round>` | no | Bounded at `maxRounds`; starts empty. |
| | `acceptedRoundNumber` | `Optional<Integer>` | yes | Populated on `EssayAccepted`. |
| | `releasedText` | `Optional<String>` | yes | Populated on `EssayReleased`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `EssayRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `EssayCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the essay reached a terminal state (`RELEASED` or `REJECTED_FINAL`). |

## Enums

`EssayStatus`: `DRAFTING`, `REFLECTING`, `ACCEPTED`, `RELEASED`, `REJECTED_FINAL`.

`ReflectorVerdict`: `ACCEPT`, `REVISE`.

## Events (`EssayEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `EssayCreated` | `topic, wordCeiling, maxRounds, createdAt` | Workflow `startStep` | → `DRAFTING` |
| `RoundDrafted` | `roundNumber, draft: EssayDraft` | After `draftStep` returns | (no status change; appends to `rounds[]`) |
| `RoundGuardrailVerdictRecorded` | `roundNumber, verdict: GuardrailVerdict` | After `guardrailStep` runs | `passed=true` → continues to `releaseStep`; `passed=false` → `DRAFTING` (policy-revise) |
| `RoundReflected` | `roundNumber, reflection: Reflection` | After `reflectStep` returns | `ACCEPT` → `ACCEPTED`; `REVISE` + rounds < max → `DRAFTING`; `REVISE` + rounds = max → `REJECTED_FINAL` |
| `EssayAccepted` | `roundNumber, acceptedText` | `Reflection.verdict = ACCEPT` | → `ACCEPTED` |
| `EssayReleased` | `releasedText, finishedAt` | Guardrail passes on the accepted draft | → `RELEASED` |
| `EssayRejectedFinal` | `bestRoundNumber, bestText, rejectionReason` | `rounds.size() == maxRounds` AND last reflection is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `ReflectionRecorded` | `roundNumber, verdict, score, guardrailPassed, recordedAt` | `EvalSampler` per reflected round; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `TopicSubmitted` | `essayId, topic, wordCeiling, requestedBy, submittedAt` |

## View row

`EssayRow` is structurally identical to `Essay` — the `rounds` list is bounded at `maxRounds` (default 4) so the row stays small enough to stream down the SSE connection without pagination. The view's `TableUpdater` consumes every `EssayEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllEssays` returning the full list. Callers filter by `status` client-side.
