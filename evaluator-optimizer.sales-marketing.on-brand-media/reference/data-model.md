# Data model — on-brand-genmedia

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CampaignBrief` | `product` | `String` | no | The product or service being promoted. |
| | `channel` | `String` | no | Distribution channel (e.g., "LinkedIn", "Instagram"). |
| | `tone` | `String` | no | Desired voice register (default: "professional"). |
| | `tokenCeiling` | `int` | no | Combined token cap across all three copy fields. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `GeneratedAsset` | `headline` | `String` | no | Ad headline; no more than 12 words. |
| | `bodyCopy` | `String` | no | Main copy block; factual, product-specific. |
| | `socialCaption` | `String` | no | Single-post caption for the specified channel; under 40 words. |
| | `tokenCount` | `int` | no | Combined token count of all three fields. |
| | `generatedAt` | `Instant` | no | When the brand agent returned the asset. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the asset. |
| | `reasonCode` | `String` | no | `OK`, `PROHIBITED_CONTENT`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ReviewNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Review` | `verdict` | `ReviewerVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `ReviewNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `reviewedAt` | `Instant` | no | When the reviewer returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `asset` | `GeneratedAsset` | no | The brand agent's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `review` | `Optional<Review>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Asset` (entity state) | `assetId` | `String` | no | Unique id. |
| | `product` | `String` | no | Product name from the brief. |
| | `channel` | `String` | no | Distribution channel. |
| | `tone` | `String` | no | Desired voice register. |
| | `tokenCeiling` | `int` | no | Per-asset combined token cap. |
| | `maxAttempts` | `int` | no | Per-asset retry ceiling (default 4). |
| | `status` | `AssetStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `AssetApproved`. |
| | `approvedAsset` | `Optional<GeneratedAsset>` | yes | Populated on `AssetApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `AssetRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `AssetCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the asset reached a terminal state. |

## Enums

`AssetStatus`: `GENERATING`, `REVIEWING`, `APPROVED`, `REJECTED_FINAL`.

`ReviewerVerdict`: `APPROVE`, `REVISE`.

## Events (`AssetEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `AssetCreated` | `product, channel, tone, tokenCeiling, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, asset: GeneratedAsset` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `REVIEWING`; `passed=false` → `GENERATING` (re-generate) |
| `AttemptReviewed` | `attemptNumber, review: Review` | After `reviewStep` returns | (no status change; populates `attempts[n].review`) |
| `AssetApproved` | `attemptNumber, approvedAsset: GeneratedAsset` | `Review.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `AssetRejectedFinal` | `bestAttemptNumber, bestAsset: GeneratedAsset, rejectionReason` | `attempts.size() == maxAttempts` AND last review is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `ReviewEvalRecorded` | `attemptNumber, verdict, score, guardrailFailed, recordedAt` | `EvalSampler` per reviewed attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`CampaignQueue`)

| Event | Payload |
|---|---|
| `BriefSubmitted` | `assetId, product, channel, tone, tokenCeiling, requestedBy, submittedAt` |

## View row

`AssetRow` is structurally identical to `Asset` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `AssetEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllAssets` returning the full list. Callers filter by `status` client-side.
