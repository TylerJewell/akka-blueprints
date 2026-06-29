# Data model

Every record the generated system defines. `Optional<T>` marks nullable lifecycle fields (Lesson 6). Akka's Jackson config serializes `Optional<T>` as the raw value or `null`.

## `Debate` — entity state and view row

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Debate UUID |
| `topic` | `Optional<String>` | yes (until start) | The debate topic |
| `status` | `DebateStatus` | no | Lifecycle status |
| `currentRound` | `int` | no | Rounds completed so far |
| `maxRounds` | `int` | no | Round cap (5) |
| `rounds` | `List<Round>` | no (may be empty) | Recorded rounds |
| `conclusion` | `Optional<String>` | yes (until synthesis) | Balanced conclusion |
| `keyArguments` | `Optional<List<String>>` | yes (until synthesis) | Key arguments from both sides |
| `qualityScore` | `Optional<Double>` | yes (until eval) | Synthesis quality score |
| `createdAt` | `Optional<Instant>` | yes (until start) | When the debate started |
| `concludedAt` | `Optional<Instant>` | yes (until conclusion) | When the synthesis was recorded |
| `failedReason` | `Optional<String>` | yes (until failure) | Why the debate failed |

`emptyState()` returns `Debate.initial("")` with placeholder values and no `commandContext()` reference (Lesson 3).

## `Round` — value record

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `number` | `int` | no | Round number (1-based) |
| `advocateArgument` | `String` | no | The Advocate's argument text |
| `criticArgument` | `String` | no | The Critic's rebuttal text |

## `Argument` — agent result (Advocate, Critic)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `position` | `String` | no | `"for"` or `"against"` |
| `text` | `String` | no | The argument text |

## `Synthesis` — agent result (Synthesizer)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `conclusion` | `String` | no | Balanced conclusion paragraph |
| `keyArguments` | `List<String>` | no | Key arguments from both sides |

## `DebateStatus` enum

`PENDING`, `DEBATING`, `SYNTHESIZING`, `CONCLUDED`, `FAILED`.

## Events

| Event | Trigger |
|---|---|
| `DebateStarted` | Workflow `startStep` records the topic and `createdAt`; status → `DEBATING` |
| `RoundRecorded` | Workflow `roundStep` records one advocate + critic round; `currentRound` increments |
| `SynthesisRecorded` | Workflow `synthesisStep` records the conclusion and key arguments; status → `CONCLUDED`, sets `concludedAt` |
| `SynthesisEvaluated` | `SynthesisEvalConsumer` records the quality score |
| `DebateFailed` | Workflow `error` step (timeout, retry exhaustion, or guardrail-blocked synthesis); status → `FAILED` |

## View row

`DebatesView` uses `Debate` as its row type. Single query `getAllDebates` (`SELECT * AS debates FROM debates_view`) — no `WHERE status` filter, because Akka cannot auto-index the enum column (Lesson 2); status filtering happens client-side in `DebateEndpoint`.

## Inbound queue

`InboundRequestQueue` emits `InboundRequestQueued(topic)`; `DebateRequestConsumer` starts one `DebateModerator` workflow with a fresh UUID per event.
