# Data model — self-eval-loop

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Brief` | `topic` | `String` | no | The user-submitted brief. |
| | `characterCeiling` | `int` | no | Per-post hard cap on draft length. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `DraftPassage` | `text` | `String` | no | The passage itself. |
| | `characterCount` | `int` | no | Length of `text`. |
| | `draftedAt` | `Instant` | no | When the Poet returned the draft. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the draft. |
| | `reasonCode` | `String` | no | `OK`, `OVER_CEILING`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `CritiqueNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `ACCEPT`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Critique` | `verdict` | `CriticVerdict` | no | `ACCEPT` or `REVISE`. |
| | `notes` | `CritiqueNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the Critic returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `draft` | `DraftPassage` | no | The Poet's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `critique` | `Optional<Critique>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Post` (entity state) | `postId` | `String` | no | Unique id. |
| | `topic` | `String` | no | Original brief. |
| | `characterCeiling` | `int` | no | Per-post cap. |
| | `maxAttempts` | `int` | no | Per-post retry ceiling (default 4). |
| | `status` | `PostStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `acceptedAttemptNumber` | `Optional<Integer>` | yes | Populated on `PostAccepted`. |
| | `acceptedText` | `Optional<String>` | yes | Populated on `PostAccepted`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `PostRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `PostCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the post reached a terminal state. |

## Enums

`PostStatus`: `DRAFTING`, `EVALUATING`, `ACCEPTED`, `REJECTED_FINAL`.

`CriticVerdict`: `ACCEPT`, `REVISE`.

## Events (`PostEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `PostCreated` | `topic, characterCeiling, maxAttempts, createdAt` | Workflow `startStep` | → `DRAFTING` |
| `AttemptDrafted` | `attemptNumber, draft: DraftPassage` | After `draftStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `EVALUATING`; `passed=false` → `DRAFTING` (re-draft) |
| `AttemptCritiqued` | `attemptNumber, critique: Critique` | After `critiqueStep` returns | (no status change; populates `attempts[n].critique`) |
| `PostAccepted` | `attemptNumber, acceptedText` | `Critique.verdict = ACCEPT` | → `ACCEPTED`, `finishedAt = now` |
| `PostRejectedFinal` | `bestAttemptNumber, bestText, rejectionReason` | `attempts.size() == maxAttempts` AND last critique is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, ceilingExceeded, recordedAt` | `EvalSampler` per critiqued attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `BriefSubmitted` | `postId, topic, characterCeiling, requestedBy, submittedAt` |

## View row

`PostRow` is structurally identical to `Post` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `PostEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllPosts` returning the full list. Callers filter by `status` client-side.
