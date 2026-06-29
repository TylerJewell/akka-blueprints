# Data model

Authoritative record forms. `/akka:implement` writes these exactly. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

## `Campaign` — event-sourced state and View row type

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Campaign id (= workflow id) |
| `brief` | `Optional<String>` | yes | The submitted project brief |
| `status` | `CampaignStatus` | no | Lifecycle status |
| `receivedAt` | `Optional<Instant>` | yes | When the brief was received |
| `findings` | `Optional<MarketFindings>` | yes | Research findings |
| `researchedAt` | `Optional<Instant>` | yes | When research recorded |
| `strategy` | `Optional<StrategyDraft>` | yes | Strategy draft |
| `strategizedAt` | `Optional<Instant>` | yes | When strategy recorded |
| `content` | `Optional<ContentArtifacts>` | yes | Content artifacts |
| `contentDraftedAt` | `Optional<Instant>` | yes | When content recorded |
| `planSummary` | `Optional<String>` | yes | Assembled plan summary |
| `assembledAt` | `Optional<Instant>` | yes | When plan assembled (G1 passed) |
| `evalScore` | `Optional<Integer>` | yes | Self-eval score 0–100 |
| `evalNotes` | `Optional<String>` | yes | Self-eval notes |
| `evaluatedAt` | `Optional<Instant>` | yes | When self-eval recorded |
| `failureReason` | `Optional<String>` | yes | Why the campaign failed |
| `failedAt` | `Optional<Instant>` | yes | When it failed |

`emptyState()` returns `Campaign.initial("")` with no `commandContext()` reference (Lesson 3).

## `CampaignStatus` enum

`RECEIVED · RESEARCHED · STRATEGIZED · CONTENT_DRAFTED · ASSEMBLED · EVALUATED · FAILED`

## `CampaignEvent` (sealed)

| Event | Trigger |
|---|---|
| `BriefReceived(String brief, Instant at)` | `receiveBrief` command |
| `ResearchRecorded(MarketFindings findings, Instant at)` | `recordResearch` after research step |
| `StrategyRecorded(StrategyDraft strategy, Instant at)` | `recordStrategy` after strategy step |
| `ContentRecorded(ContentArtifacts content, Instant at)` | `recordContent` after content step |
| `PlanAssembled(String summary, Instant at)` | `assemblePlan` after G1 passes |
| `PlanEvaluated(int score, String notes, Instant at)` | `recordEvaluation` after eval step |
| `CampaignFailed(String reason, Instant at)` | `fail` on step failover |

## Worker result records

| Record | Fields |
|---|---|
| `MarketFindings` | `List<String> claims`, `List<String> sources` |
| `StrategyDraft` | `String positioning`, `List<String> channels`, `List<String> tactics` |
| `ContentArtifacts` | `List<String> taglines`, `List<String> posts` |
| `CampaignPlan` | `String summary` |
| `PlanEval` | `int score`, `String notes` |
| `SearchResult` | `String title`, `String snippet`, `String source` |

## `BriefQueue`

State records each inbound brief; single command `enqueueBrief(String brief)` emits `BriefEnqueued(String brief, Instant at)`. `BriefConsumer` reacts to `BriefEnqueued` and starts a `StrategyWorkflow`.

## View row

`CampaignsView` row type is `Campaign` (above). One query `getAllCampaigns` returns `SELECT * AS campaigns FROM campaigns_view`; a `streamAllCampaigns` query backs the SSE endpoint. No `WHERE status` filter — the enum column cannot be auto-indexed (Lesson 2); callers filter client-side.
