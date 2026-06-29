# SPEC — vto-genmedia

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GenMedia for Commerce.
**One-line pitch:** A shopper submits a garment and a model image reference; one `VtoAgent` walks the request through three task phases — **PREPARE** asset metadata, **GENERATE** a try-on composite and video clip, **VALIDATE** the output against safety rules — with each phase gated on the prior phase's recorded output and the safety guardrail blocking unsafe media before it reaches the data layer.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales-and-marketing e-commerce domain. One `VtoAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PREPARE task's typed output becomes the GENERATE task's instruction context; the GENERATE task's typed output becomes the VALIDATE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- An **`after-llm-response` guardrail** sits between the agent and its output on the GENERATE task. After the LLM response is produced and before the typed `MediaResult` is returned to the workflow, `ImageSafetyGuardrail` inspects the generated composite image and video frame for unsafe content categories (explicit imagery, minors, hate symbols, graphic violence). A flagged output is rejected with a structured `safety-violation` reason returned to the agent loop. The agent retries within its 4-iteration budget, generating an alternate output. On accept, the guardrail records an `OutputCleared` audit entry on `TryOnEntity`. On final rejection (budget exhausted), the entity transitions to `FAILED` and the guardrail reason is visible in the UI's rejection log.

The blueprint shows that in a media-generation pipeline the `after-llm-response` hook is the right cut: the safety check runs on the fully-assembled output — not on intermediate tool calls — so it catches composite artefacts that no single tool call could reveal.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **garment** from the seeded catalogue (or uploads a garment image reference URL) and selects a **model preset** (`standing-front`, `standing-side`, `seated`). Optionally checks **Include video clip**.
2. The user clicks **Run try-on**. The UI POSTs to `/api/tryons` and receives a `tryOnId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PREPARING` — the workflow has started `prepareStep` and the agent has been handed the PREPARE task.
4. Within ~10–20 s the card reaches `ASSETS_READY` — the `AssetBundle` is visible in the card detail (garment metadata, model preset, background colour swatch, target dimensions). The agent's PREPARE task returned; the workflow recorded `AssetsResolved` and advanced to `generateStep`.
5. Within ~20–40 s the card reaches `MEDIA_GENERATED`. The composite image thumbnail appears in the right pane; if video was requested, a short clip player placeholder appears. The `MediaResult` is visible (image URL, video URL if present, generation parameters used).
6. Within ~10 s the card reaches `VALIDATED`. The safety check passed; the right pane now shows the full `ValidatedMedia` — composite image, optional video clip, safety verdict badge — plus a quality score chip (1–5) and a one-line rationale.
7. The user can submit another request; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TryOnEndpoint` | `HttpEndpoint` | `/api/tryons/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `TryOnEntity`, `TryOnView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TryOnEntity` | `EventSourcedEntity` | Per-request lifecycle: created → preparing → assets-ready → generating → media-generated → validating → validated → evaluated. Source of truth. | `TryOnEndpoint`, `VtoPipelineWorkflow` | `TryOnView` |
| `VtoPipelineWorkflow` | `Workflow` | One workflow per try-on request. Steps: `prepareStep` → `generateStep` → `validateStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `TryOnEndpoint` after `CREATED` | `VtoAgent`, `TryOnEntity` |
| `VtoAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `VtoTasks.java`: `PREPARE_ASSETS` → `AssetBundle`, `GENERATE_MEDIA` → `MediaResult`, `VALIDATE_OUTPUT` → `ValidatedMedia`. Each task is registered with the phase-appropriate function tools. | invoked by `VtoPipelineWorkflow` | returns typed results |
| `PrepareTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `resolveGarment(garmentId)` and `resolveModelPreset(preset)`. Reads from `src/main/resources/sample-data/garments/*.json` and `presets/*.json`. | called from PREPARE task | returns `GarmentSpec` / `ModelPreset` |
| `GenerateTools` | function-tools class | Implements `compositeImage(garmentSpec, modelPreset, backgroundSpec)` and `renderVideoClip(assetBundle)`. Returns deterministic sample assets from `src/main/resources/sample-data/generated/`. | called from GENERATE task | returns `ImageAsset` / `VideoAsset` |
| `ValidateTools` | function-tools class | Implements `checkAspectRatio(imageAsset)` and `checkColourFidelity(imageAsset, garmentSpec)`. Deterministic in-process checks. | called from VALIDATE task | returns `DimensionReport` / `ColourReport` |
| `ImageSafetyGuardrail` | `after-llm-response` guardrail (registered on `VtoAgent`) | Inspects the generated `MediaResult` after the LLM response is produced and before it is returned to the workflow. Rejects outputs that contain unsafe content. | every LLM response from GENERATE task | accept / structured-reject |
| `RenderQualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `ValidatedMedia`, `AssetBundle`. Output: `QualityResult{score, rationale}`. | called from `evalStep` | returns score |
| `TryOnView` | `View` | Read model: one row per try-on request for the UI. | `TryOnEntity` events | `TryOnEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record GarmentSpec(
    String garmentId,
    String displayName,
    String category,      // "top" | "bottom" | "dress" | "outerwear"
    String primaryColour,
    String imageUrl
) {}

record ModelPreset(
    String presetId,
    String pose,          // "standing-front" | "standing-side" | "seated"
    String backgroundColour,
    int targetWidthPx,
    int targetHeightPx
) {}

record AssetBundle(
    GarmentSpec garment,
    ModelPreset preset,
    boolean includeVideo,
    Instant preparedAt
) {}

record ImageAsset(
    String assetId,
    String url,
    int widthPx,
    int heightPx,
    String mimeType
) {}

record VideoAsset(
    String assetId,
    String url,
    int durationMs,
    String mimeType
) {}

record MediaResult(
    ImageAsset composite,
    Optional<VideoAsset> videoClip,
    Map<String, String> generationParams,
    Instant generatedAt
) {}

record DimensionReport(
    boolean aspectRatioOk,
    String expected,
    String actual
) {}

record ColourReport(
    boolean colourFidelityOk,
    double deltaE,         // CIE76 colour difference; threshold 10.0
    String dominantActual,
    String expectedPrimary
) {}

record ValidatedMedia(
    ImageAsset composite,
    Optional<VideoAsset> videoClip,
    DimensionReport dimensionReport,
    ColourReport colourReport,
    String safetyVerdict,  // "CLEARED" | "FLAGGED"
    Instant validatedAt
) {}

record QualityResult(
    int score,             // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record SafetyRejection(
    String category,       // "EXPLICIT" | "MINORS" | "HATE" | "VIOLENCE"
    String reason,
    Instant rejectedAt
) {}

record TryOnRecord(
    String tryOnId,
    String garmentId,
    String modelPreset,
    boolean includeVideo,
    Optional<AssetBundle> assets,
    Optional<MediaResult> media,
    Optional<ValidatedMedia> validatedMedia,
    Optional<QualityResult> quality,
    TryOnStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<SafetyRejection> safetyRejections
) {}

enum TryOnStatus {
    CREATED, PREPARING, ASSETS_READY, GENERATING, MEDIA_GENERATED,
    VALIDATING, VALIDATED, EVALUATED, FAILED
}
```

Events on `TryOnEntity`: `TryOnCreated`, `PrepareStarted`, `AssetsResolved`, `GenerateStarted`, `MediaGenerated`, `ValidateStarted`, `MediaValidated`, `EvaluationScored`, `SafetyRejected`, `TryOnFailed`.

Every nullable lifecycle field on the `TryOnRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tryons` — body `{ garmentId, modelPreset, includeVideo }` → `{ tryOnId }`.
- `GET /api/tryons` — list all try-on requests, newest-first.
- `GET /api/tryons/{id}` — one try-on record.
- `GET /api/tryons/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: GenMedia for Commerce</title>`.

The App UI tab is a two-column layout: a left rail with the live list of try-on requests (status pill + garment name + age) and a right pane with the selected request's detail — asset summary, generated image thumbnail, optional video clip player, validation reports, quality score chip, and a safety-rejection log strip if any guardrail rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `after-llm-response` guardrail (image safety)**: `ImageSafetyGuardrail` is registered on `VtoAgent` and runs after every LLM response produced during the GENERATE task. The guardrail inspects the `MediaResult`'s composite image (and video frame if present) for four unsafe content categories: explicit imagery, depiction of minors, hate symbols, and graphic violence. The accept rule is: all four category checks return clean. On reject, the guardrail returns a structured `safety-violation` error to the agent loop with the offending category, and records a `SafetyRejected{category, reason}` event on `TryOnEntity` for audit visibility. The agent loop retries within its 4-iteration budget. If the budget is exhausted, the workflow's error step fires and the entity transitions to `FAILED`.

- **E1 — `on-decision-eval`**: runs immediately after `MediaValidated` lands, as `evalStep` inside the workflow. `RenderQualityScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): it checks that aspect ratio is within tolerance (DimensionReport.aspectRatioOk), that colour fidelity meets the threshold (ColourReport.deltaE ≤ 10.0), that a composite image asset is present, and that a video clip asset is present when `includeVideo = true`. Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `VtoAgent` → `prompts/vto-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded garment `summer-dress-blue` with preset `standing-front` and `includeVideo = true`; within 60 s the request reaches `EVALUATED` with a non-null composite image, a non-null video clip, dimension and colour reports both passing, and a quality score chip showing 5.
2. **J2** — The agent's GENERATE task returns a composite image the safety guardrail flags as unsafe (mock LLM path). `ImageSafetyGuardrail` rejects the output; a `SafetyRejected` event lands on the entity; the agent retries; the request eventually reaches `EVALUATED` with a clean composite. The UI's rejection-log strip shows the one rejected output.
3. **J3** — A request whose mock-LLM trajectory produces a composite where `ColourReport.deltaE > 10.0` is scored 3 or lower with a rationale naming the colour fidelity gap; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-request trace (logged at `INFO`); the PREPARE task's log shows only PREPARE-tool calls, the GENERATE task's log shows only GENERATE-tool calls, the VALIDATE task's log shows only VALIDATE-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named vto-genmedia demonstrating the sequential-pipeline x sales-marketing cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-sales-marketing-vto-genmedia. Java package
io.akka.samples.genmediaforcommerce. Akka 3.6.0. HTTP port 9809.

Components to wire (exactly):

- 1 AutonomousAgent VtoAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/vto-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(4))
  entries — one per declared Task. Function tools are registered with .tools(...) — the PREPARE,
  GENERATE, and VALIDATE tool sets are ALL registered on the agent; safety gating is the job of
  ImageSafetyGuardrail, NOT of conditional .tools(...) wiring. The after-llm-response guardrail
  (ImageSafetyGuardrail) is registered on the agent via the agent's guardrail-configuration
  block. On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow VtoPipelineWorkflow per tryOnId with four steps:
  * prepareStep — emits PrepareStarted on the entity, then calls componentClient
    .forAutonomousAgent(VtoAgent.class, "vto-agent-" + tryOnId).runSingleTask(
      TaskDef.instructions("GarmentId: " + garmentId + "\nPreset: " + modelPreset +
        "\nIncludeVideo: " + includeVideo + "\nPhase: PREPARE\nResolve garment spec and model
        preset into a typed AssetBundle.")
        .metadata("tryOnId", tryOnId)
        .metadata("phase", "PREPARE")
        .taskType(VtoTasks.PREPARE_ASSETS)
    ). Reads forTask(taskId).result(PREPARE_ASSETS) to get AssetBundle. Writes
    TryOnEntity.recordAssets(assetBundle). WorkflowSettings.stepTimeout 60s.
  * generateStep — emits GenerateStarted, then runSingleTask with TaskDef.instructions
    (formatGenerateContext(assetBundle)) and metadata.phase = "GENERATE", taskType
    GENERATE_MEDIA. Writes TryOnEntity.recordMedia(mediaResult). stepTimeout 120s (generation
    is the slowest step).
  * validateStep — emits ValidateStarted, then runSingleTask with TaskDef.instructions
    (formatValidateContext(mediaResult, assetBundle)) and metadata.phase = "VALIDATE", taskType
    VALIDATE_OUTPUT. Writes TryOnEntity.recordValidated(validatedMedia). stepTimeout 60s.
  * evalStep — runs the deterministic RenderQualityScorer over (validatedMedia, assetBundle)
    and writes TryOnEntity.recordEvaluation(quality). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(VtoPipelineWorkflow::error). The error step writes TryOnFailed
  and ends.

- 1 EventSourcedEntity TryOnEntity (one per tryOnId). State TryOnRecord{tryOnId, garmentId,
  modelPreset, includeVideo, assets: Optional<AssetBundle>, media: Optional<MediaResult>,
  validatedMedia: Optional<ValidatedMedia>, quality: Optional<QualityResult>, status:
  TryOnStatus, createdAt: Instant, finishedAt: Optional<Instant>, safetyRejections:
  List<SafetyRejection>}. TryOnStatus enum: CREATED, PREPARING, ASSETS_READY, GENERATING,
  MEDIA_GENERATED, VALIDATING, VALIDATED, EVALUATED, FAILED. Events: TryOnCreated{garmentId,
  modelPreset, includeVideo}, PrepareStarted, AssetsResolved{assets}, GenerateStarted,
  MediaGenerated{media}, ValidateStarted, MediaValidated{validatedMedia},
  EvaluationScored{quality}, SafetyRejected{category, reason, rejectedAt}, TryOnFailed{reason}.
  Commands: create, startPrepare, recordAssets, startGenerate, recordMedia, startValidate,
  recordValidated, recordEvaluation, recordSafetyRejection, fail, getTryOn. emptyState() returns
  TryOnRecord.initial("") with all Optional fields as Optional.empty() and safetyRejections =
  List.of() and no commandContext() reference (Lesson 3).

- 1 View TryOnView with row type TryOnRow that mirrors TryOnRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes TryOnEntity events. ONE query
  getAllTryOns: SELECT * AS tryOns FROM tryon_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * TryOnEndpoint at /api with POST /tryons (body {garmentId, modelPreset, includeVideo}; mints
    tryOnId; calls TryOnEntity.create(...); then starts VtoPipelineWorkflow with id
    "vto-pipeline-" + tryOnId; returns {tryOnId}), GET /tryons (list from getAllTryOns, sorted
    newest-first), GET /tryons/{id} (one row), GET /tryons/sse (Server-Sent Events forwarded
    from the view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- VtoTasks.java declaring three Task<R> constants:
    PREPARE_ASSETS = Task.name("Prepare assets").description("Resolve garment spec and model
      preset into a typed AssetBundle").resultConformsTo(AssetBundle.class);
    GENERATE_MEDIA = Task.name("Generate media").description("Produce a composite try-on image
      and optional video clip from the AssetBundle").resultConformsTo(MediaResult.class);
    VALIDATE_OUTPUT = Task.name("Validate output").description("Run dimension and colour
      fidelity checks on the generated media").resultConformsTo(ValidatedMedia.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- PrepareTools.java — @FunctionTool resolveGarment(String garmentId) -> GarmentSpec reading from
  src/main/resources/sample-data/garments/<garmentId>.json; @FunctionTool
  resolveModelPreset(String preset) -> ModelPreset reading from
  src/main/resources/sample-data/presets/<preset>.json.

- GenerateTools.java — @FunctionTool compositeImage(GarmentSpec garment, ModelPreset preset,
  String backgroundColour) -> ImageAsset reading the deterministic composite from
  src/main/resources/sample-data/generated/<garmentId>-<presetId>.png (or returning a
  placeholder ImageAsset with a synthetic url when the file is absent); @FunctionTool
  renderVideoClip(AssetBundle bundle) -> VideoAsset reading the deterministic clip from
  src/main/resources/sample-data/generated/<garmentId>-<presetId>.mp4 (or returning a
  placeholder VideoAsset).

- ValidateTools.java — @FunctionTool checkAspectRatio(ImageAsset image, ModelPreset preset) ->
  DimensionReport (checks image widthPx/heightPx match preset targetWidthPx/targetHeightPx
  within a 5% tolerance); @FunctionTool checkColourFidelity(ImageAsset image, GarmentSpec
  garment) -> ColourReport (deterministic deltaE computation against the garment primaryColour
  field; threshold 10.0 for pass).

- ImageSafetyGuardrail.java — implements the after-llm-response hook. Inspects the
  MediaResult's composite ImageAsset url and optional VideoAsset url for unsafe content
  categories (EXPLICIT, MINORS, HATE, VIOLENCE). Deterministic in the sample: the presence of
  the literal substring "unsafe-fixture" in the url triggers a rejection for testing. On reject,
  calls TryOnEntity.recordSafetyRejection(category, reason, rejectedAt) and returns
  Guardrail.reject("safety-violation: <category> — <reason>"). On accept, records an implicit
  "CLEARED" audit path (no event needed for the happy path; SafetyRejected only records
  rejections).

- RenderQualityScorer.java — pure deterministic logic (no LLM). Inputs: ValidatedMedia,
  AssetBundle. Outputs: QualityResult with score and rationale. Four checks, one point per check
  satisfied, starting from a base of 1: aspect ratio OK (dimensionReport.aspectRatioOk),
  colour fidelity OK (colourReport.colourFidelityOk), composite image present (non-null url),
  video clip present when includeVideo = true. Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9809 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/garments.jsonl with 5 seeded garment lines covering at
  least two categories.

- src/main/resources/sample-data/garments/*.json — five files each carrying a GarmentSpec entry
  with deterministic content.

- src/main/resources/sample-data/presets/*.json — three files: standing-front, standing-side,
  seated. Each carrying a ModelPreset entry.

- src/main/resources/sample-data/generated/ — placeholder image and video clip files keyed by
  garmentId+presetId, used by GenerateTools deterministic paths.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root with sector = retail-ecommerce, data.data_classes.pii =
  false (garment and pose selection; no person-level PII stored), decisions.authority_level =
  recommend-only (the try-on composite is advisory to the purchase decision),
  oversight.human_in_loop = true (shopper views the composite before purchasing),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "unsafe-generated-image", "colour-fidelity-gap",
  "dimension-mismatch", "video-clip-missing"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/vto-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: GenMedia for Commerce", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left = live list
  of try-on request cards; right = selected-request detail with garment header, asset summary,
  composite image thumbnail, optional video clip player, dimension and colour fidelity badges,
  quality score chip, safety-rejection log strip). Browser title exactly:
  <title>Akka Sample: GenMedia for Commerce</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/<task-id>.json,
  picks one entry pseudo-randomly per call (seedFor(tryOnId)), and deserialises into the task's
  typed return.
- Per-task mock-response shapes for THIS blueprint:
    prepare-assets.json — 5 AssetBundle entries, one per seeded garment, each with a
      GarmentSpec and a ModelPreset resolved. tool_calls contain resolveGarment + resolveModelPreset.
    generate-media.json — 5 MediaResult entries paired one-to-one with the AssetBundle entries.
      Each carries an ImageAsset and optional VideoAsset with deterministic placeholder urls.
      tool_calls contain compositeImage + renderVideoClip. Plus 1 deliberately UNSAFE entry
      whose ImageAsset url contains "unsafe-fixture" — the guardrail rejects it; the mock then
      falls through to a normal entry. The mock should select the unsafe entry on the FIRST
      iteration of every 3rd try-on (modulo seed) so J2 is reproducible.
    validate-output.json — 5 ValidatedMedia entries. Each carries DimensionReport and
      ColourReport. Plus 1 deliberately COLOUR-FAIL entry whose ColourReport.deltaE > 10.0,
      paired with a low quality score — J3 verifies this.
- A MockModelProvider.seedFor(tryOnId) helper makes per-request selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md.
Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. VtoAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion VtoTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (prepareStep
  60s, generateStep 120s, validateStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the TryOnRecord row record is Optional<T>.
- Lesson 7: VtoTasks.java with PREPARE_ASSETS, GENERATE_MEDIA, VALIDATE_OUTPUT constants is
  mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9809 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (VtoAgent). The
  on-decision eval is rule-based (RenderQualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the after-llm-response guardrail (ImageSafetyGuardrail) is the runtime mechanism that
  enforces the safety contract. Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: prepareStep writes AssetBundle onto the
  entity, generateStep reads it and builds the GENERATE task's instruction context from it,
  validateStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
