# Data model — shared-memory-multi-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BlockSeed` | `blockId` | `String` | no | Id assigned at creation. |
| | `blockName` | `String` | no | Human-readable block name (e.g., "project-glossary"). |
| | `initialContent` | `String` | no | Seed content for the first writer cycle. |
| | `seededBy` | `String` | no | UI identifier of the submitter. |
| `MemoryFragment` | `fragmentId` | `String` | no | Deterministic id `blockId + "-f-" + authorId + "-" + cycle`. |
| | `blockId` | `String` | no | The block this fragment belongs to. |
| | `authorId` | `String` | no | Writer agent instance id. |
| | `content` | `String` | no | The knowledge contribution (post-sanitization). |
| | `status` | `FragmentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When submitted. |
| | `sanitizedNote` | `Optional<String>` | yes | Description of what the sanitizer redacted, if anything. |
| `ConsolidatedSnapshot` | `blockId` | `String` | no | The block this snapshot covers. |
| | `content` | `String` | no | Merged block content. |
| | `mergeSummary` | `String` | no | One-sentence description of what changed in this pass. |
| | `mergedFragmentIds` | `List<String>` | no | All fragment ids incorporated. |
| | `consolidatedAt` | `Instant` | no | When the snapshot was committed. |
| `WriterContribution` | `blockId` | `String` | no | The block the writer is contributing to. |
| | `snapshotContent` | `String` | no | The current block snapshot passed to the agent. |
| | `authorId` | `String` | no | Writer agent instance id. |

## Entity state — `MemoryBlock` (`MemoryBlockEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `blockId` | `String` | no | Unique id. |
| `blockName` | `String` | no | Human-readable name. |
| `currentContent` | `String` | no | The latest consolidated or seed content. |
| `status` | `BlockStatus` | no | See enum. |
| `pendingFragmentCount` | `int` | no | How many fragments are currently `PENDING` or `SANITIZED`. |
| `lastConsolidationSummary` | `Optional<String>` | yes | Summary from the most recent consolidation pass. |
| `lastConsolidatedAt` | `Optional<Instant>` | yes | When the last consolidation was committed. |
| `lastWriterAgent` | `Optional<String>` | yes | The id of the most recent writer agent that contributed a fragment. |
| `createdAt` | `Instant` | no | When `BlockSeeded` emitted. |
| `updatedAt` | `Optional<Instant>` | yes | When the block was last modified. |

## Entity state — `Fragment` (`FragmentEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `fragmentId` | `String` | no | Deterministic id. |
| `blockId` | `String` | no | Owning block. |
| `authorId` | `String` | no | Writer agent instance id. |
| `content` | `String` | no | Fragment body (may be redacted if sanitized). |
| `status` | `FragmentStatus` | no | See enum. |
| `sanitized` | `boolean` | no | Whether the sanitizer made any substitutions. |
| `sanitizedNote` | `Optional<String>` | yes | What was redacted. |
| `createdAt` | `Instant` | no | When submitted. |
| `mergedAt` | `Optional<Instant>` | yes | When marked `MERGED`. |

## Enums

`BlockStatus`: `SEEDED`, `ACTIVE`, `CONSOLIDATING`, `CONSOLIDATED`, `ARCHIVED`.
`FragmentStatus`: `PENDING`, `SANITIZED`, `MERGED`, `REJECTED`, `REQUEUED`.

## Events — `MemoryBlockEntity`

| Event | Payload | Transition |
|---|---|---|
| `BlockSeeded` | `blockId, blockName, initialContent, seededBy, createdAt` | → SEEDED |
| `BlockActivated` | `blockId, activatedAt` | SEEDED → ACTIVE (on first fragment increment) |
| `ConsolidationStarted` | `blockId, startedAt` | ACTIVE → CONSOLIDATING |
| `BlockConsolidated` | `blockId, content, mergeSummary, consolidatedAt` | CONSOLIDATING → CONSOLIDATED |
| `BlockArchived` | `blockId, archivedAt` | ACTIVE/CONSOLIDATED → ARCHIVED |

## Events — `FragmentEntity`

| Event | Payload | Transition |
|---|---|---|
| `FragmentSubmitted` | `fragmentId, blockId, authorId, content, createdAt` | → PENDING |
| `FragmentSanitized` | `fragmentId, redactedContent, sanitizedNote, sanitizedAt` | PENDING → SANITIZED |
| `FragmentMerged` | `fragmentId, mergedAt` | PENDING/SANITIZED → MERGED |
| `FragmentRejected` | `fragmentId, reason, rejectedAt` | (any active) → REJECTED |
| `FragmentRequeued` | `fragmentId, requeuedAt` | PENDING → REQUEUED |

## Events — `MemoryEventLog`

| Event | Payload |
|---|---|
| `BlockCreated` | `blockId, blockName, seededBy, createdAt` |
| `ConsolidationTriggered` | `blockId, triggeredBy, triggeredAt` |

## Key-value state — `AgentRegistry`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `agents` | `Map<String, AgentEntry>` | no | Keyed by agent id. |
| `AgentEntry.agentId` | `String` | no | Writer instance id. |
| `AgentEntry.displayName` | `String` | no | Human-readable label. |
| `AgentEntry.registeredAt` | `Instant` | no | When the agent was registered. |
| `AgentEntry.lastActiveAt` | `Optional<Instant>` | yes | Updated on each `WriteWorkflow` cycle. |

## View rows

`BlockRow` mirrors `MemoryBlock` but omits the large `currentContent` field — it keeps `blockName`, `status`, `pendingFragmentCount`, `lastConsolidationSummary`, `lastConsolidatedAt`, `lastWriterAgent`, and `createdAt` only, so the SSE stream stays small. Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6).

`FragmentRow` mirrors `Fragment` in full — all fields. The fragment panel reads this row type for the detail view.
