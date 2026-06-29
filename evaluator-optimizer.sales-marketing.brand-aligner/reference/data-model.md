# Data model — brand-aligner

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CampaignBrief` | `topic` | `String` | no | The user-submitted campaign brief. |
| | `targetAudience` | `String` | no | Intended audience segment. |
| | `wordCeiling` | `int` | no | Per-material hard cap on variant word count. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `CopyVariant` | `text` | `String` | no | The marketing copy itself. |
| | `wordCount` | `int` | no | Word count of `text`. |
| | `generatedAt` | `Instant` | no | When the Copywriter returned the variant. |
| `ComplianceVerdict` | `passed` | `boolean` | no | Whether the compliance check accepted the variant. |
| | `reasonCode` | `String` | no | `OK`, `OVER_WORD_CEILING`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ReviewNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `BrandReview` | `verdict` | `ReviewerVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `ReviewNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `reviewedAt` | `Instant` | no | When the Reviewer returned. |
| `VariantAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes compliance-blocked attempts). |
| | `variant` | `CopyVariant` | no | The Copywriter's output for this attempt. |
| | `compliance` | `ComplianceVerdict` | no | The deterministic check's verdict. |
| | `review` | `Optional<BrandReview>` | yes | Empty when the compliance check blocked; populated otherwise. |
| `Material` (entity state) | `materialId` | `String` | no | Unique id. |
| | `topic` | `String` | no | Original campaign brief. |
| | `targetAudience` | `String` | no | Audience segment. |
| | `wordCeiling` | `int` | no | Per-material cap. |
| | `maxAttempts` | `int` | no | Per-material retry ceiling (default 4). |
| | `status` | `MaterialStatus` | no | See enum. |
| | `attempts` | `List<VariantAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `MaterialApproved`. |
| | `approvedText` | `Optional<String>` | yes | Populated on `MaterialApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `MaterialRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `MaterialCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the material reached a terminal state. |

## Enums

`MaterialStatus`: `DRAFTING`, `REVIEWING`, `APPROVED`, `REJECTED_FINAL`.

`ReviewerVerdict`: `APPROVE`, `REVISE`.

## Events (`MaterialEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `MaterialCreated` | `topic, targetAudience, wordCeiling, maxAttempts, createdAt` | Workflow `startStep` | → `DRAFTING` |
| `VariantGenerated` | `attemptNumber, variant: CopyVariant` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `ComplianceVerdictRecorded` | `attemptNumber, verdict: ComplianceVerdict` | After `complianceStep` | `passed=true` → `REVIEWING`; `passed=false` → `DRAFTING` (re-generate) |
| `VariantReviewed` | `attemptNumber, review: BrandReview` | After `reviewStep` returns | (no status change; populates `attempts[n].review`) |
| `MaterialApproved` | `attemptNumber, approvedText` | `BrandReview.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `MaterialRejectedFinal` | `bestAttemptNumber, bestText, rejectionReason` | `attempts.size() == maxAttempts` AND last review is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `BrandEvalRecorded` | `attemptNumber, verdict, score, wordCeilingExceeded, recordedAt` | `EvalSampler` per reviewed attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`BriefQueue`)

| Event | Payload |
|---|---|
| `BriefSubmitted` | `materialId, topic, targetAudience, wordCeiling, requestedBy, submittedAt` |

## View row

`MaterialRow` is structurally identical to `Material` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `MaterialEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllMaterials` returning the full list. Callers filter by `status` client-side.
