# Data model — team-poems-multi-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WritingPrompt` | `promptId` | `String` | no | Id assigned at submission. |
| | `title` | `String` | no | Short poem title or working name. |
| | `inspiration` | `String` | no | Free-text prompt note. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `StanzaSpec` | `title` | `String` | no | Short descriptive stanza title. |
| | `verseBrief` | `String` | no | What the stanza should convey and at what register. |
| | `form` | `VerseForm` | no | Verse form the poet must follow. |
| | `dependsOn` | `List<String>` | no | Titles of stanzas in the same plan that must finish first (may be empty). |
| `VersePlan` | `planNote` | `String` | no | One-sentence structural approach from the poetry director. |
| | `stanzas` | `List<StanzaSpec>` | no | Three to five stanza specs. |
| `DraftLine` | `lineNumber` | `int` | no | 1-based position within the stanza. |
| | `text` | `String` | no | Line of verse (no placeholder content). |
| `CoordinationRequest` | `toPoet` | `String` | no | Poet id the request is addressed to. |
| | `question` | `String` | no | Concrete question about rhythm or tonal continuity. |
| `DraftVerse` | `stanzaId` | `String` | no | The stanza this draft completes. |
| | `lines` | `List<DraftLine>` | no | Two to six lines (empty when a coordination request is raised). |
| | `approach` | `String` | no | One-sentence description of the tonal or formal choice. |
| | `coordinationRequest` | `Optional<CoordinationRequest>` | yes | Present only when the poet is blocked on a peer. |
| `QualityReport` | `passed` | `boolean` | no | Whether the draft passed the quality gate. |
| | `issues` | `List<String>` | no | One string per failing line/check (empty on pass). |
| | `ranAt` | `Instant` | no | When the gate ran. |
| `CoordinationMessage` | `messageId` | `String` | no | Unique id. |
| | `fromPoet` | `String` | no | Sender poet id. |
| | `toPoet` | `String` | no | Recipient poet id. |
| | `stanzaId` | `String` | no | The blocked stanza the message concerns. |
| | `question` | `String` | no | The coordination question. |
| | `sentAt` | `Instant` | no | When posted. |
| | `reply` | `Optional<String>` | yes | Populated on reply. |
| | `repliedAt` | `Optional<Instant>` | yes | Populated on reply. |

## Entity state — `Stanza` (`StanzaEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `stanzaId` | `String` | no | Deterministic id `poemId + "-s" + index`. |
| `poemId` | `String` | no | Owning poem. |
| `title` | `String` | no | From the `StanzaSpec`. |
| `verseBrief` | `String` | no | From the `StanzaSpec`. |
| `form` | `VerseForm` | no | From the `StanzaSpec`. |
| `dependsOn` | `List<String>` | no | Dependency titles. |
| `status` | `StanzaStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Poet that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `approachNote` | `Optional<String>` | yes | Tonal/formal note from the submitted draft. |
| `lineCount` | `Optional<Integer>` | yes | Number of lines in the draft. |
| `qualityReport` | `Optional<QualityReport>` | yes | Latest quality-gate result. |
| `blockedReason` | `Optional<String>` | yes | Why the stanza is blocked. |
| `completedAt` | `Optional<Instant>` | yes | When the stanza reached `DONE`. |
| `createdAt` | `Instant` | no | When `StanzaCreated` emitted. |

## Entity state — `Poem` (`PoemEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `poemId` | `String` | no | Unique id. |
| `title` | `String` | no | Poem title. |
| `inspiration` | `String` | no | Original prompt note. |
| `submittedBy` | `String` | no | Submitter id. |
| `status` | `PoemStatus` | no | See enum. |
| `stanzaIds` | `List<String>` | no | Ids of stanzas created from the plan. |
| `planNote` | `Optional<String>` | yes | Populated on `PoemAssigned`. |
| `createdAt` | `Instant` | no | When `PoemCreated` emitted. |
| `publishedAt` | `Optional<Instant>` | yes | When the poem reached `PUBLISHED`. |

## Enums

`StanzaStatus`: `OPEN`, `CLAIMED`, `DRAFTING`, `IN_REVIEW`, `DONE`, `BLOCKED`.
`PoemStatus`: `RECEIVED`, `ASSIGNED`, `DRAFTING`, `PUBLISHED`.
`VerseForm`: `HAIKU`, `FREE_VERSE`, `RHYMING_COUPLET`, `SONNET_QUATRAIN`.

## Events — `StanzaEntity`

| Event | Payload | Transition |
|---|---|---|
| `StanzaCreated` | `stanzaId, poemId, title, verseBrief, form, dependsOn, createdAt` | → OPEN |
| `StanzaClaimed` | `poetId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `DraftingStarted` | `startedAt` | CLAIMED → DRAFTING |
| `VerseSubmitted` | `approachNote, lineCount, submittedAt` | DRAFTING → IN_REVIEW |
| `QualityPassed` | `qualityReport` | IN_REVIEW → DONE, `completedAt = now` |
| `QualityFailed` | `qualityReport` | IN_REVIEW → DRAFTING (retry) |
| `StanzaBlocked` | `reason, blockedAt` | (any active) → BLOCKED |
| `StanzaReleased` | `releasedAt` | CLAIMED/DRAFTING → OPEN, clears `claimedBy` |
| `StanzaCompleted` | `completedAt` | terminal marker emitted alongside `QualityPassed` |

Note: `QualityPassed` carries the `DONE` transition and `completedAt`; `StanzaCompleted` is the explicit terminal event the poem-completion check subscribes to. Prefer two events for a clean audit trail.

## Events — `PoemEntity`

| Event | Payload | Transition |
|---|---|---|
| `PoemCreated` | `poemId, title, inspiration, submittedBy, createdAt` | → RECEIVED |
| `PoemAssigned` | `stanzaIds, planNote` | RECEIVED → ASSIGNED |
| `PoemDrafting` | `startedAt` | ASSIGNED → DRAFTING |
| `PoemPublished` | `publishedAt` | DRAFTING → PUBLISHED |

## Events — `CoordinationMailbox`

| Event | Payload |
|---|---|
| `CoordinationPosted` | `CoordinationMessage` |
| `CoordinationReplied` | `messageId, reply, repliedAt` |

## Events — `PromptQueue`

| Event | Payload |
|---|---|
| `PromptSubmitted` | `promptId, title, inspiration, submittedBy, submittedAt` |

## Key-value state — `EditorialControl`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `paused` | `boolean` | no | Whether the team is frozen. |
| `pausedReason` | `Optional<String>` | yes | Operator-supplied reason. |
| `pausedBy` | `Optional<String>` | yes | Operator id. |
| `pausedAt` | `Optional<Instant>` | yes | When paused. |

## View row

`StanzaRow` mirrors `Stanza` but drops the heavy `DraftLine` contents — it keeps `approachNote` and `lineCount` only, so the SSE stream stays small. Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6). The draft lines themselves are not projected; the UI fetches only the summary fields.
