# Data model — sleeptime-consolidation

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MemoryBlockContent` | `blockId` | `String` | no | Block this content belongs to. |
| | `sessionId` | `String` | no | Session that produced this content. |
| | `rawContent` | `String` | no | Accumulated raw interaction text, not yet consolidated. |
| | `topicHints` | `List<String>` | no | Advisory topic tags from the caller. |
| | `capturedAt` | `Instant` | no | When this raw content was captured. |
| `ConsolidatedContent` | `consolidatedText` | `String` | no | Compact consolidated representation (≤ 400 chars). |
| | `changeRationale` | `String` | no | One sentence: what changed from the prior version. |
| | `priorVersion` | `int` | no | Version number of the block before this consolidation. |
| | `consolidatedAt` | `Instant` | no | When the consolidation was written. |
| `DriftResult` | `driftScore` | `int` | no | Semantic drift 0–100. |
| | `rationale` | `String` | no | One sentence naming the strongest drift signal. |
| | `priorSummary` | `String` | no | ≤ 30-word paraphrase of the prior version. |
| | `currentSummary` | `String` | no | ≤ 30-word paraphrase of the current version. |
| `AgentResponse` | `reply` | `String` | no | Answer to the user's prompt, ≤ 150 words. |
| | `blockIdRead` | `String` | no | Which block was read. |
| | `blockVersionRead` | `int` | no | Version of the block that was read. |
| | `respondedAt` | `Instant` | no | When the agent finished. |
| `GuardrailRecord` | `callerId` | `String` | no | Identity of the caller that was blocked. |
| | `toolName` | `String` | no | The tool call that was rejected. |
| | `rejectionReason` | `String` | no | Structured rejection message. |
| | `triggeredAt` | `Instant` | no | When the guardrail fired. |
| `SessionCounter` | `sessionId` | `String` | no | Session identity. |
| | `totalSteps` | `int` | no | Cumulative step count since session creation. |
| | `stepsSinceLastConsolidation` | `int` | no | Steps since the last `CounterReset`. |
| | `lastConsolidatedAt` | `Instant` | no | When `CounterReset` last fired. |
| `MemoryBlock` (entity state) | `blockId` | `String` | no | Block identity. |
| | `sessionId` | `String` | no | Owning session. |
| | `rawContent` | `Optional<MemoryBlockContent>` | yes | Latest raw content; populated after `BlockUpdated`. |
| | `consolidated` | `Optional<ConsolidatedContent>` | yes | Latest consolidated form; populated after `BlockConsolidated`. |
| | `version` | `int` | no | Monotonically incrementing; starts at 0. |
| | `drift` | `Optional<DriftResult>` | yes | Latest drift score; populated after `DriftScored`. |
| | `lastGuardrailTrip` | `Optional<GuardrailRecord>` | yes | Most recent guardrail trip; populated after `GuardrailTriggered`. |
| | `status` | `BlockStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `BlockCreated` was emitted. |
| | `lastConsolidatedAt` | `Optional<Instant>` | yes | When `BlockConsolidated` was last emitted. |

## Enums

`BlockStatus`: `ACTIVE`, `CONSOLIDATING`, `CONSOLIDATED`, `STALE`.

## Events (`MemoryBlockEntity`)

| Event | Payload | Transition |
|---|---|---|
| `BlockCreated` | `blockId, sessionId, rawContent, topicHints` | → ACTIVE |
| `BlockUpdated` | `rawContent, topicHints` | CONSOLIDATED → ACTIVE (or stays ACTIVE); version++ |
| `ConsolidationStarted` | `workflowId, startedAt` | ACTIVE → CONSOLIDATING |
| `BlockConsolidated` | `consolidated, newVersion` | CONSOLIDATING → CONSOLIDATED |
| `BlockMarkedStale` | `reason` | CONSOLIDATING → STALE (terminal) |
| `DriftScored` | `driftScore, rationale, priorSummary, currentSummary` | (no status change; populates `drift`) |
| `GuardrailTriggered` | `callerId, toolName, rejectionReason` | (no status change; populates `lastGuardrailTrip`) |

## Events (`InteractionCounterEntity`)

| Event | Payload |
|---|---|
| `StepRecorded` | `sessionId, newTotal: int, newStepsSinceConsolidation: int` |
| `CounterReset` | `sessionId, resetAt: Instant` |

## View row

`MemoryBlockRow` mirrors `MemoryBlock` but omits `rawContent.rawContent` (the raw text is not projected into the view — fetched on-demand via `GET /api/memory/blocks/{blockId}`). The row includes a derived `consolidatedSummary` field (first 120 chars of `consolidated.consolidatedText`) for list display. `stepsSinceLastConsolidation` is denormalised from `InteractionCounterEntity` into the row.
