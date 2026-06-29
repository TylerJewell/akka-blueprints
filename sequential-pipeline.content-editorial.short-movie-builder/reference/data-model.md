# Data model — short-movie-agents

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Scene` | `sceneId` | `String` | no | Short slug. Unique within a `MovieScript`. |
| | `description` | `String` | no | Prose description of the scene's visual action. |
| | `dialogueLine` | `String` | no | Single line of dialogue or narration for the scene. |
| | `type` | `SceneType` | no | One of `INTRO`, `ACTION`, `DIALOGUE`, `TRANSITION`, `OUTRO`. |
| `MovieScript` | `scenes` | `List<Scene>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `genre` | `String` | no | Short genre label (e.g. `training`, `product-teaser`). |
| | `writtenAt` | `Instant` | no | When the SCRIPT task returned. |
| `Shot` | `shotId` | `String` | no | `"s-" + sha1(sceneId)[0:8]`. |
| | `sceneId` | `String` | no | MUST equal a `Scene.sceneId` from the upstream `MovieScript`. |
| | `framing` | `String` | no | Camera framing label (e.g. `wide-shot`, `close-up`). |
| | `durationSeconds` | `int` | no | At least 2. |
| `Storyboard` | `shots` | `List<Shot>` | no | One per `Scene`. Possibly empty. |
| | `designedAt` | `Instant` | no | When the STORYBOARD task returned. |
| `PackageScene` | `packageSceneId` | `String` | no | Short stable id (e.g. `"ps-001"`). |
| | `sceneId` | `String` | no | MUST equal a `Scene.sceneId` from the upstream `MovieScript`. |
| | `shotId` | `String` | no | MUST equal a `Shot.shotId` from the upstream `Storyboard`. |
| | `assetPlaceholder` | `String` | no | `"asset-<shotId>-<sceneId>"`. |
| | `orderIndex` | `int` | no | Ascending, no gaps. |
| `AssembledPackage` | `packageScenes` | `List<PackageScene>` | no | One per `Shot`. |
| | `totalRuntimeSeconds` | `int` | no | Sum of `shot.durationSeconds` for all shots. |
| | `assembledAt` | `Instant` | no | When the ASSEMBLE task returned. |
| `CoherenceCheck` | `sceneId` | `String` | no | MUST equal a `Scene.sceneId` from the upstream `MovieScript`. |
| | `coherent` | `boolean` | no | `true` iff the paired `Shot.sceneId` equals `scene.sceneId`. |
| | `note` | `String` | no | Short explanation ("shot sceneId matches" or name of mismatch). |
| `ReviewResult` | `coherenceChecks` | `List<CoherenceCheck>` | no | One per `Scene`. |
| | `summary` | `String` | no | 1–3 sentences. "All scenes coherent." or names incoherent scenes. |
| | `reviewedAt` | `Instant` | no | When the REVIEW task returned. |
| `CoherenceResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `PackageScorer` finished. |
| `SafetyBlock` | `phase` | `String` | no | `SCRIPT` / `STORYBOARD` / `ASSEMBLE` / `REVIEW`. |
| | `violationType` | `String` | no | `explicit-content` / `regulated-brand` / `restricted-claim` / `incoherent-violence`. |
| | `reason` | `String` | no | Structured reason from `ContentSafetyGuard`. |
| | `blockedAt` | `Instant` | no | When the guard fired. |
| `MovieRecord` (entity state) | `productionId` | `String` | no | — |
| | `brief` | `Optional<String>` | yes | Populated after `ProductionCreated`. |
| | `script` | `Optional<MovieScript>` | yes | Populated after `ScriptWritten`. |
| | `storyboard` | `Optional<Storyboard>` | yes | Populated after `StoryboardDesigned`. |
| | `assembledPackage` | `Optional<AssembledPackage>` | yes | Populated after `PackageAssembled`. |
| | `reviewResult` | `Optional<ReviewResult>` | yes | Populated after `ReviewCompleted`. |
| | `coherenceResult` | `Optional<CoherenceResult>` | yes | Populated after `CoherenceScoredEvent`. |
| | `status` | `ProductionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ProductionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `safetyBlocks` | `List<SafetyBlock>` | no | Appended on every `SafetyBlockRecorded` event; empty on the happy path. |

Every nullable lifecycle field on `MovieRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ProductionStatus`: `CREATED`, `SCRIPTING`, `SCRIPTED`, `STORYBOARDING`, `STORYBOARDED`, `ASSEMBLING`, `ASSEMBLED`, `REVIEWING`, `REVIEWED`, `FAILED`.

`SceneType`: `INTRO`, `ACTION`, `DIALOGUE`, `TRANSITION`, `OUTRO`.

`Phase` (used in `SafetyBlock` and `ContentSafetyGuard` annotations): `SCRIPT`, `STORYBOARD`, `ASSEMBLE`, `REVIEW`.

## Events (`MovieEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ProductionCreated` | `brief: String` | → CREATED |
| `ScriptingStarted` | — | → SCRIPTING |
| `ScriptWritten` | `script: MovieScript` | → SCRIPTED |
| `StoryboardingStarted` | — | → STORYBOARDING |
| `StoryboardDesigned` | `storyboard: Storyboard` | → STORYBOARDED |
| `AssemblingStarted` | — | → ASSEMBLING |
| `PackageAssembled` | `assembledPackage: AssembledPackage` | → ASSEMBLED |
| `ReviewingStarted` | — | → REVIEWING |
| `ReviewCompleted` | `reviewResult: ReviewResult` | → REVIEWED (happy) |
| `CoherenceScoredEvent` | `coherenceResult: CoherenceResult` | no status change (appends score) |
| `SafetyBlockRecorded` | `phase, violationType, reason, blockedAt` | no status change (audit-only) |
| `ProductionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `MovieRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `safetyBlocks = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`MovieRow` mirrors `MovieRecord` exactly. The UI fetches the full row via `GET /api/productions/{id}` and streams updates via `GET /api/productions/sse`.

The view declares ONE query: `getAllProductions: SELECT * AS productions FROM movie_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`MovieTasks.java`)

```java
public final class MovieTasks {
  public static final Task<MovieScript> WRITE_SCRIPT = Task
      .name("Write script")
      .description("Generate a MovieScript with scenes and dialogue from the creative brief")
      .resultConformsTo(MovieScript.class);

  public static final Task<Storyboard> DESIGN_STORYBOARD = Task
      .name("Design storyboard")
      .description("Plan shots and framings for each scene in the MovieScript")
      .resultConformsTo(Storyboard.class);

  public static final Task<AssembledPackage> ASSEMBLE_PACKAGE = Task
      .name("Assemble package")
      .description("Build a renderable AssembledPackage by pairing shots with scenes")
      .resultConformsTo(AssembledPackage.class);

  public static final Task<ReviewResult> REVIEW_PACKAGE = Task
      .name("Review package")
      .description("Check scene coherence and produce a ReviewResult summarising any gaps")
      .resultConformsTo(ReviewResult.class);

  private MovieTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## PackageScorer checks

`PackageScorer` is deterministic and makes no LLM call. Inputs: `AssembledPackage`, `Storyboard`, `MovieScript`. Four checks, one point each, on a base of 1:

1. **Scene coverage** — every `Scene.sceneId` in `MovieScript` has a corresponding `PackageScene.sceneId` in `AssembledPackage`.
2. **Shot reference validity** — every `PackageScene.shotId` appears in `Storyboard.shots[].shotId`.
3. **Runtime plausibility** — `totalRuntimeSeconds > 0` AND `totalRuntimeSeconds >= 0.5 * scenes.size() * 3`.
4. **Order consistency** — `packageScenes` are ordered by `orderIndex` ascending with no gaps (i.e., `packageScenes[i].orderIndex == i`).

Score range 1–5. Rationale names the first failing check, or "all checks passed" for a score of 5.
