# Data model — story-teller

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `StyleConstraints` | `tone` | `Tone` | no | Enum value controlling narrative register. |
| | `wordCountTarget` | `int` | no | Target word count for the story body. |
| | `pointOfView` | `PointOfView` | no | Enum value: FIRST or THIRD person. |
| `StoryRequest` | `storyId` | `String` | no | UUID minted by `StoryEndpoint`. |
| | `genre` | `String` | no | User-selected genre (Fantasy / Mystery / Sci-Fi / Romance / Horror / Custom). |
| | `rawPrompt` | `String` | no | Pre-enrichment prompt text. Audit-only. |
| | `constraints` | `StyleConstraints` | no | Submitted style parameters. |
| | `requestedBy` | `String` | no | User identifier. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `EnrichedPrompt` | `safePrompt` | `String` | yes | Safety-passed prompt; null if blocked. |
| | `contentTags` | `List<String>` | no | e.g. `["mystery","crime","locked-room"]`. |
| | `safetyPassed` | `boolean` | no | True = agent may proceed; false = blocked. |
| | `safetyReason` | `String` | yes | Non-null only when `safetyPassed == false`. |
| `GeneratedStory` | `title` | `String` | no | Short evocative title (3–10 words). |
| | `body` | `String` | no | Narrative body (at least 150 characters). |
| | `authorNote` | `String` | no | Craft rationale (at least 30 characters). |
| | `genre` | `String` | no | Must match submitted genre. |
| | `generatedAt` | `Instant` | no | When the agent returned. |
| `QualityResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `Story` (entity state) | `storyId` | `String` | no | — |
| | `request` | `Optional<StoryRequest>` | yes | Populated after `StoryRequested`. |
| | `enriched` | `Optional<EnrichedPrompt>` | yes | Populated after `PromptEnriched` or `StoryBlocked`. |
| | `story` | `Optional<GeneratedStory>` | yes | Populated after `StoryRecorded`. |
| | `quality` | `Optional<QualityResult>` | yes | Populated after `QualityScored`. |
| | `status` | `StoryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `StoryRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Story` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Tone`: `WHIMSICAL`, `GRITTY`, `LITERARY`, `NEUTRAL`.
`PointOfView`: `FIRST`, `THIRD`.
`StoryStatus`: `REQUESTED`, `ENRICHED`, `BLOCKED`, `GENERATING`, `STORY_RECORDED`, `SCORED`, `FAILED`.

## Events (`StoryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `StoryRequested` | `request` | → REQUESTED |
| `PromptEnriched` | `enriched` | → ENRICHED |
| `StoryBlocked` | `reason: String` | → BLOCKED (terminal) |
| `GenerationStarted` | — | → GENERATING |
| `StoryRecorded` | `story` | → STORY_RECORDED |
| `QualityScored` | `quality` | → SCORED (terminal happy) |
| `StoryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Story.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`StoryRow` mirrors `Story` minus `request.rawPrompt` (the audit log keeps that). The UI always displays the raw prompt from the in-memory state — this domain has no PII concern — but the view omits it to keep the read model lean.

The view declares ONE query: `getAllStories: SELECT * AS stories FROM story_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`StoryTasks.java`)

```java
public final class StoryTasks {
  public static final Task<GeneratedStory> GENERATE_STORY = Task
      .name("Generate story")
      .description("Read the attached enriched prompt and produce a GeneratedStory matching the style constraints")
      .resultConformsTo(GeneratedStory.class);

  private StoryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
