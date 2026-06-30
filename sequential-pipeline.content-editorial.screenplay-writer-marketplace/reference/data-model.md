# Data model — screenplay-writer-marketplace

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Character` | `characterId` | `String` | no | Stable short id (`ch-NN`). |
| | `placeholder` | `String` | no | `[NAME_N]` token assigned by `PiiSanitizer`. |
| | `archetype` | `String` | no | Brief role description (e.g., "project lead"). |
| `Setting` | `settingId` | `String` | no | Stable short id (`s-NN`). |
| | `placeholder` | `String` | no | Location reference as it appears in sanitized text. |
| | `type` | `String` | no | `INT` or `EXT`. |
| `Beat` | `beatId` | `String` | no | Stable short id (`b-NN`). |
| | `description` | `String` | no | One-sentence summary. Uses `[NAME_N]` placeholders. |
| | `dramatic_function` | `String` | no | E.g., inciting incident, confrontation, resolution. |
| `ParsedSource` | `characters` | `List<Character>` | no | Possibly empty (J6 demonstrates the empty path). |
| | `settings` | `List<Setting>` | no | Possibly empty. |
| | `beats` | `List<Beat>` | no | Possibly empty. |
| | `parsedAt` | `Instant` | no | When the PARSE task returned. |
| `SceneOutline` | `sceneId` | `String` | no | Stable short id (`sc-NN`). |
| | `slugline` | `String` | no | Uppercase INT./EXT. line. |
| | `beatIds` | `List<String>` | no | Each entry MUST equal a `Beat.beatId` from the upstream `ParsedSource`. |
| | `characterIds` | `List<String>` | no | Each entry MUST equal a `Character.characterId` from the upstream `ParsedSource`. |
| | `settingId` | `String` | no | MUST equal a `Setting.settingId` from the upstream `ParsedSource`. |
| `ScenePlan` | `scenes` | `List<SceneOutline>` | no | Possibly empty. |
| | `developedAt` | `Instant` | no | When the DEVELOP task returned. |
| `DialogueLine` | `characterId` | `String` | no | MUST equal a `Character.characterId` from the upstream `ParsedSource`. |
| | `line` | `String` | no | The character's spoken line. |
| `SceneBlock` | `sceneId` | `String` | no | MUST equal a `SceneOutline.sceneId` from the `ScenePlan`. |
| | `slugline` | `String` | no | MUST equal the corresponding `SceneOutline.slugline`. |
| | `action` | `String` | no | Present-tense action paragraph. Must not contain raw PII. |
| | `dialogue` | `List<DialogueLine>` | no | May be empty for silent scenes. |
| `Screenplay` | `title` | `String` | no | 1-line title. |
| | `logline` | `String` | no | 1-sentence logline. |
| | `scenes` | `List<SceneBlock>` | no | `scenes.size() == scenePlan.scenes.size()`. |
| | `formattedAt` | `Instant` | no | When the FORMAT task returned. |
| `PlaceholderMap` | `originalToPlaceholder` | `Map<String,String>` | no | Maps each original PII token to its placeholder. Never projected to `ScreenplayView`. |
| `SanitizedSource` | `sanitizedText` | `String` | no | Source text with all PII replaced by placeholders. |
| | `placeholderMap` | `PlaceholderMap` | no | The mapping used during sanitization. |
| `DeliveryCheckResult` | `passed` | `boolean` | no | `true` if no PII detected; `false` if blocked. |
| | `detectedTokens` | `List<String>` | no | Empty on pass. Populated with raw PII values on block. |
| | `checkedAt` | `Instant` | no | When `DeliveryGuardrail` finished. |
| `ScreenplayRecord` (entity state) | `screenplayId` | `String` | no | — |
| | `sourceTitle` | `Optional<String>` | yes | Populated after `ScreenplayCreated`. |
| | `parsedSource` | `Optional<ParsedSource>` | yes | Populated after `SourceParsed`. |
| | `scenePlan` | `Optional<ScenePlan>` | yes | Populated after `ScenesDevoped`. |
| | `screenplay` | `Optional<Screenplay>` | yes | Populated after `ScreenplayFormatted`. |
| | `deliveryCheck` | `Optional<DeliveryCheckResult>` | yes | Populated after `ScreenplayDelivered` or `DeliveryBlocked`. |
| | `status` | `ScreenplayStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ScreenplayCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `ScreenplayRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6). The `placeholderMap` is stored on the entity but is NOT projected onto `ScreenplayRow` and is never returned by any API endpoint.

## Enums

`ScreenplayStatus`: `CREATED`, `PARSING`, `PARSED`, `DEVELOPING`, `DEVELOPED`, `FORMATTING`, `FORMATTED`, `DELIVERED`, `DELIVERY_BLOCKED`, `FAILED`.

## Events (`ScreenplayEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ScreenplayCreated` | `sourceTitle: String` | → CREATED |
| `ParseStarted` | — | → PARSING |
| `SourceParsed` | `parsedSource: ParsedSource` | → PARSED |
| `DevelopStarted` | — | → DEVELOPING |
| `ScenesDevoped` | `scenePlan: ScenePlan` | → DEVELOPED |
| `FormatStarted` | — | → FORMATTING |
| `ScreenplayFormatted` | `screenplay: Screenplay` | → FORMATTED |
| `ScreenplayDelivered` | `deliveryCheck: DeliveryCheckResult` | → DELIVERED (terminal happy) |
| `DeliveryBlocked` | `detectedTokens: List<String>, reason: String` | → DELIVERY_BLOCKED (terminal) |
| `ScreenplayFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ScreenplayRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ScreenplayRow` mirrors `ScreenplayRecord` exactly, minus the private `placeholderMap` field. The UI fetches the full row via `GET /api/screenplays/{id}` and streams updates via `GET /api/screenplays/sse`.

The view declares ONE query: `getAllScreenplays: SELECT * AS screenplays FROM screenplay_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`ScreenplayTasks.java`)

```java
public final class ScreenplayTasks {
  public static final Task<ParsedSource> PARSE_SOURCE = Task
      .name("Parse source")
      .description("Extract characters, settings, and beats from sanitized source text")
      .resultConformsTo(ParsedSource.class);

  public static final Task<ScenePlan> DEVELOP_SCENES = Task
      .name("Develop scenes")
      .description("Build a scene outline and assign beats to scenes")
      .resultConformsTo(ScenePlan.class);

  public static final Task<Screenplay> FORMAT_SCREENPLAY = Task
      .name("Format screenplay")
      .description("Produce production-ready scene blocks with sluglines, action, and dialogue")
      .resultConformsTo(Screenplay.class);

  private ScreenplayTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## PiiSanitizer token categories

| Category | Pattern | Placeholder prefix |
|---|---|---|
| Email address | RFC-5322 email regex | `[EMAIL_N]` |
| Phone number | E.164 format (`+` followed by 7–15 digits) | `[PHONE_N]` |
| Display name | `Name <email>` header pattern — the `Name` segment | `[NAME_N]` |

Counter `N` resets per screenplay. The same token within one screenplay always maps to the same placeholder (stable across multiple uses in the source text). The mapping is stored in `PlaceholderMap.originalToPlaceholder` and held privately on `ScreenplayEntity` — never projected to `ScreenplayView`.
