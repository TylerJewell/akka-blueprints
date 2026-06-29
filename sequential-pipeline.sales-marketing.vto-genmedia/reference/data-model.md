# Data model — vto-genmedia

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GarmentSpec` | `garmentId` | `String` | no | Catalogue identifier. |
| | `displayName` | `String` | no | Human-readable name shown in the UI. |
| | `category` | `String` | no | One of `top`, `bottom`, `dress`, `outerwear`. |
| | `primaryColour` | `String` | no | CSS hex string (e.g. `#1E90FF`). Used by `ColourReport`. |
| | `imageUrl` | `String` | no | Canonical URL of the garment catalogue image. |
| `ModelPreset` | `presetId` | `String` | no | Slug identifying the preset. |
| | `pose` | `String` | no | One of `standing-front`, `standing-side`, `seated`. |
| | `backgroundColour` | `String` | no | CSS hex string for the composite background. |
| | `targetWidthPx` | `int` | no | Target composite image width in pixels. |
| | `targetHeightPx` | `int` | no | Target composite image height in pixels. |
| `AssetBundle` | `garment` | `GarmentSpec` | no | Resolved garment from the PREPARE phase. |
| | `preset` | `ModelPreset` | no | Resolved model preset from the PREPARE phase. |
| | `includeVideo` | `boolean` | no | Whether the GENERATE phase should call `renderVideoClip`. |
| | `preparedAt` | `Instant` | no | When the PREPARE task returned. |
| `ImageAsset` | `assetId` | `String` | no | Short stable id (`img-<4 hex>`). |
| | `url` | `String` | no | URL of the generated composite image. |
| | `widthPx` | `int` | no | Actual width; must be within 5% of `preset.targetWidthPx`. |
| | `heightPx` | `int` | no | Actual height; must be within 5% of `preset.targetHeightPx`. |
| | `mimeType` | `String` | no | MIME type (typically `image/png`). |
| `VideoAsset` | `assetId` | `String` | no | Short stable id (`vid-<4 hex>`). |
| | `url` | `String` | no | URL of the generated video clip. |
| | `durationMs` | `int` | no | Clip duration in milliseconds. |
| | `mimeType` | `String` | no | MIME type (typically `video/mp4`). |
| `MediaResult` | `composite` | `ImageAsset` | no | Generated composite image. |
| | `videoClip` | `Optional<VideoAsset>` | yes | Present only when `AssetBundle.includeVideo` is `true`. |
| | `generationParams` | `Map<String,String>` | no | Key–value pairs recording model, background colour, etc. |
| | `generatedAt` | `Instant` | no | When the GENERATE task returned. |
| `DimensionReport` | `aspectRatioOk` | `boolean` | no | `true` if actual dimensions are within 5% of target. |
| | `expected` | `String` | no | Target ratio string (e.g. `"2:3"`). |
| | `actual` | `String` | no | Actual ratio string computed from `ImageAsset`. |
| `ColourReport` | `colourFidelityOk` | `boolean` | no | `true` if `deltaE ≤ 10.0`. |
| | `deltaE` | `double` | no | CIE76 colour difference between dominant actual and expected primary. |
| | `dominantActual` | `String` | no | Dominant colour extracted from the composite (CSS hex). |
| | `expectedPrimary` | `String` | no | `GarmentSpec.primaryColour`. |
| `ValidatedMedia` | `composite` | `ImageAsset` | no | Same `ImageAsset` as in `MediaResult`. |
| | `videoClip` | `Optional<VideoAsset>` | yes | Carried forward from `MediaResult`. |
| | `dimensionReport` | `DimensionReport` | no | Output of `checkAspectRatio`. |
| | `colourReport` | `ColourReport` | no | Output of `checkColourFidelity`. |
| | `safetyVerdict` | `String` | no | `"CLEARED"` or `"FLAGGED"`. |
| | `validatedAt` | `Instant` | no | When the VALIDATE task returned. |
| `QualityResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `RenderQualityScorer` finished. |
| `SafetyRejection` | `category` | `String` | no | `EXPLICIT` / `MINORS` / `HATE` / `VIOLENCE`. |
| | `reason` | `String` | no | Structured reason from `ImageSafetyGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `TryOnRecord` (entity state) | `tryOnId` | `String` | no | — |
| | `garmentId` | `String` | no | From the original request. |
| | `modelPreset` | `String` | no | From the original request. |
| | `includeVideo` | `boolean` | no | From the original request. |
| | `assets` | `Optional<AssetBundle>` | yes | Populated after `AssetsResolved`. |
| | `media` | `Optional<MediaResult>` | yes | Populated after `MediaGenerated`. |
| | `validatedMedia` | `Optional<ValidatedMedia>` | yes | Populated after `MediaValidated`. |
| | `quality` | `Optional<QualityResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `TryOnStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TryOnCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `safetyRejections` | `List<SafetyRejection>` | no | Appended on every `SafetyRejected` event; empty on the happy path. |

Every nullable lifecycle field on `TryOnRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TryOnStatus`: `CREATED`, `PREPARING`, `ASSETS_READY`, `GENERATING`, `MEDIA_GENERATED`, `VALIDATING`, `VALIDATED`, `EVALUATED`, `FAILED`.

## Events (`TryOnEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TryOnCreated` | `garmentId, modelPreset, includeVideo` | → CREATED |
| `PrepareStarted` | — | → PREPARING |
| `AssetsResolved` | `assets: AssetBundle` | → ASSETS_READY |
| `GenerateStarted` | — | → GENERATING |
| `MediaGenerated` | `media: MediaResult` | → MEDIA_GENERATED |
| `ValidateStarted` | — | → VALIDATING |
| `MediaValidated` | `validatedMedia: ValidatedMedia` | → VALIDATED |
| `EvaluationScored` | `quality: QualityResult` | → EVALUATED (terminal happy) |
| `SafetyRejected` | `category, reason, rejectedAt` | no status change (audit-only) |
| `TryOnFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `TryOnRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `safetyRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`TryOnRow` mirrors `TryOnRecord` exactly. The UI fetches the full row via `GET /api/tryons/{id}` and streams updates via `GET /api/tryons/sse`.

The view declares ONE query: `getAllTryOns: SELECT * AS tryOns FROM tryon_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`VtoTasks.java`)

```java
public final class VtoTasks {
  public static final Task<AssetBundle> PREPARE_ASSETS = Task
      .name("Prepare assets")
      .description("Resolve garment spec and model preset into a typed AssetBundle")
      .resultConformsTo(AssetBundle.class);

  public static final Task<MediaResult> GENERATE_MEDIA = Task
      .name("Generate media")
      .description("Produce a composite try-on image and optional video clip from the AssetBundle")
      .resultConformsTo(MediaResult.class);

  public static final Task<ValidatedMedia> VALIDATE_OUTPUT = Task
      .name("Validate output")
      .description("Run dimension and colour fidelity checks on the generated media")
      .resultConformsTo(ValidatedMedia.class);

  private VtoTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Safety guardrail inspection path

`ImageSafetyGuardrail` fires on every LLM response from the GENERATE task. It inspects `MediaResult.composite.url` and, when present, `MediaResult.videoClip.url`. In the sample implementation, the literal substring `"unsafe-fixture"` in either URL triggers a rejection — making the guardrail path exercisable without a live content-moderation API. A production deployer replaces this check with a real moderation API call. The guardrail does not inspect tool call inputs; it operates on the assembled output only.
