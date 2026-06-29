# SPEC — short-movie-agents

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Short Movie Builder.
**One-line pitch:** A user submits a creative brief; one `MovieAgent` carries it through four production phases — **SCRIPT** a screenplay, **STORYBOARD** the shots, **ASSEMBLE** a renderable package, **REVIEW** for coherence — with each phase gated on the prior phase's recorded output and a content-safety guardrail inspecting every LLM response before it is committed.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain. One `MovieAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the SCRIPT task's typed output becomes the STORYBOARD task's instruction context; the STORYBOARD task's typed output becomes the ASSEMBLE task's instruction context; the ASSEMBLE task's typed output becomes the REVIEW task's instruction context. The agent never holds all four phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- An **`after-llm-response` guardrail** sits between the agent and the workflow's result-commit path. After each phase's LLM response returns (before the workflow writes the corresponding event onto `MovieEntity`), `ContentSafetyGuard` inspects the text payload for policy violations: explicit content, regulated brand names, legally restricted claims, and incoherent violence. A violation triggers a structured `safety-block` rejection returned to the agent loop so the task can retry within its iteration budget. The same guard fires on all four phases — SCRIPT, STORYBOARD, ASSEMBLE, and REVIEW — because any phase can produce user-visible text.

The blueprint shows that sequential-pipeline governance is not only about phase ordering — a single `after-llm-response` hook applied once per phase result is the minimal intervention point for content safety, without requiring per-tool inspection.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **creative brief** into the input (or picks one of three seeded briefs — `a 60-second workplace-safety training video`, `a product teaser for a fictional smartwatch`, `a nature documentary short about urban foxes`).
2. The user clicks **Produce movie**. The UI POSTs to `/api/productions` and receives a `productionId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `SCRIPTING` — the workflow has started `scriptStep` and the agent has been handed the SCRIPT task.
4. Within ~15–25 s the card reaches `STORYBOARDING` — the typed `MovieScript` is visible in the card detail (a small table of scenes with dialogue and action lines). The agent's SCRIPT task returned; the workflow recorded `ScriptWritten` and ran the STORYBOARD task.
5. Within ~15–25 s more the card reaches `ASSEMBLING`. The `Storyboard` is visible (shot list with framing, duration, and scene references).
6. Within ~15–25 s more the card reaches `REVIEWING`. The `AssembledPackage` is visible (ordered list of `PackageScene` items each with a shot reference and an asset placeholder).
7. Within ~10 s more the card reaches `REVIEWED`. The right pane now shows the full typed `MoviePackage` — title, genre, a per-scene `PackageScene` table, and a coherence score chip (1–5) with a one-line rationale.
8. The user can submit another brief; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MovieEndpoint` | `HttpEndpoint` | `/api/productions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `MovieEntity`, `MovieView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `MovieEntity` | `EventSourcedEntity` | Per-production lifecycle: created → scripting → scripted → storyboarding → storyboarded → assembling → assembled → reviewing → reviewed. Source of truth. | `MovieEndpoint`, `MovieProductionWorkflow` | `MovieView` |
| `MovieProductionWorkflow` | `Workflow` | One workflow per production. Steps: `scriptStep` → `storyboardStep` → `assembleStep` → `reviewStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `MovieEndpoint` after `CREATED` | `MovieAgent`, `MovieEntity` |
| `MovieAgent` | `AutonomousAgent` | The single agent. Declares four `Task<R>` constants in `MovieTasks.java`: `WRITE_SCRIPT` → `MovieScript`, `DESIGN_STORYBOARD` → `Storyboard`, `ASSEMBLE_PACKAGE` → `AssembledPackage`, `REVIEW_PACKAGE` → `ReviewResult`. Each task is registered with the phase-appropriate function tools. | invoked by `MovieProductionWorkflow` | returns typed results |
| `ScriptTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `generateScenes(brief)` and `writeDialogueLine(scene)`. Reads from `src/main/resources/sample-data/briefs/*.json` for deterministic offline output. | called from SCRIPT task | returns `List<Scene>` |
| `StoryboardTools` | function-tools class | Implements `planShot(scene)` and `selectFraming(shot)`. Pure in-memory transformations. | called from STORYBOARD task | returns `List<Shot>` |
| `AssemblyTools` | function-tools class | Implements `buildPackageScene(shot, scene)` and `computeRuntime(shots)`. | called from ASSEMBLE task | returns `PackageScene` / `Duration` |
| `ReviewTools` | function-tools class | Implements `checkSceneCoherence(scene, shot)` and `generateReviewSummary(scenes)`. | called from REVIEW task | returns `CoherenceCheck` / `String` |
| `ContentSafetyGuard` | `after-llm-response` guardrail (registered on `MovieAgent`) | Inspects every phase output for policy violations before the workflow commits the result. Rejects violations with a structured `safety-block` error; records `SafetyBlockRecorded` on the entity for audit. | every task result on every phase | accept / structured-reject |
| `PackageScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `AssembledPackage`, `Storyboard`, `MovieScript`. Output: `CoherenceResult{score, rationale}`. | called from `reviewStep` | returns score |
| `MovieView` | `View` | Read model: one row per production for the UI. | `MovieEntity` events | `MovieEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Scene(String sceneId, String description, String dialogueLine, SceneType type) {}

enum SceneType { INTRO, ACTION, DIALOGUE, TRANSITION, OUTRO }

record MovieScript(List<Scene> scenes, String genre, Instant writtenAt) {}

record Shot(String shotId, String sceneId, String framing, int durationSeconds) {}

record Storyboard(List<Shot> shots, Instant designedAt) {}

record PackageScene(
    String packageSceneId,
    String sceneId,
    String shotId,
    String assetPlaceholder,
    int orderIndex
) {}

record AssembledPackage(
    List<PackageScene> packageScenes,
    int totalRuntimeSeconds,
    Instant assembledAt
) {}

record CoherenceCheck(String sceneId, boolean coherent, String note) {}

record ReviewResult(
    List<CoherenceCheck> coherenceChecks,
    String summary,
    Instant reviewedAt
) {}

record CoherenceResult(
    int score,         // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record SafetyBlock(
    String phase,
    String violationType,
    String reason,
    Instant blockedAt
) {}

record MovieRecord(
    String productionId,
    Optional<String> brief,
    Optional<MovieScript> script,
    Optional<Storyboard> storyboard,
    Optional<AssembledPackage> assembledPackage,
    Optional<ReviewResult> reviewResult,
    Optional<CoherenceResult> coherenceResult,
    ProductionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<SafetyBlock> safetyBlocks
) {}

enum ProductionStatus {
    CREATED, SCRIPTING, SCRIPTED, STORYBOARDING, STORYBOARDED,
    ASSEMBLING, ASSEMBLED, REVIEWING, REVIEWED, FAILED
}
```

Events on `MovieEntity`: `ProductionCreated`, `ScriptingStarted`, `ScriptWritten`, `StoryboardingStarted`, `StoryboardDesigned`, `AssemblingStarted`, `PackageAssembled`, `ReviewingStarted`, `ReviewCompleted`, `CoherenceScoredEvent`, `SafetyBlockRecorded`, `ProductionFailed`.

Every nullable lifecycle field on the `MovieRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/productions` — body `{ brief }` → `{ productionId }`.
- `GET /api/productions` — list all productions, newest-first.
- `GET /api/productions/{id}` — one production.
- `GET /api/productions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Short Movie Builder</title>`.

The App UI tab is a two-column layout: a left rail with the live list of productions (status pill + brief + age) and a right pane with the selected production's detail — brief, script scenes table, storyboard shot list, assembled package scenes, coherence score chip, and a safety-block log strip if any content-safety rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — `after-llm-response` guardrail (content safety)**: `ContentSafetyGuard` is registered on `MovieAgent` and runs after every LLM response, before the workflow writes the result onto `MovieEntity`. It evaluates the text payload of the completed task against four content-safety rules: no explicit or graphic content, no regulated brand names used as fictional entities, no legally restricted claims about real persons, no incoherent depiction of violence. The check is deterministic and rule-based (no additional LLM call — the guard reads the response text directly). On violation, the guard returns a structured `safety-block` error to the agent loop and calls `MovieEntity.recordSafetyBlock(phase, violationType, reason)` so the block is visible in the UI's safety-block log strip and in the audit log. The agent loop retries within its 4-iteration budget. On accept, the workflow proceeds to write the event.

## 9. Agent prompts

- `MovieAgent` → `prompts/movie-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded brief `a 60-second workplace-safety training video`; within 80 s the production reaches `REVIEWED` with a non-empty script, ≥ 3 shots, ≥ 3 package scenes, and a coherence score chip on the card.
2. **J2** — The agent's first iteration produces a script containing a policy-violating phrase (mock LLM path). `ContentSafetyGuard` rejects the response; a `SafetyBlockRecorded` event lands on the entity; the agent retries; the production eventually completes correctly. The UI's safety-block log strip shows the one blocked response.
3. **J3** — A production whose mock-LLM trajectory produces a `PackageScene` referencing a `shotId` absent from the `Storyboard` is scored 1 with a rationale naming the missing shot reference; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-production trace (logged at `INFO`); the SCRIPT task's log shows only SCRIPT-tool calls, the STORYBOARD task's log shows only STORYBOARD-tool calls, the ASSEMBLE task's log shows only ASSEMBLE-tool calls, the REVIEW task's log shows only REVIEW-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named short-movie-agents demonstrating the sequential-pipeline x
content-editorial cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact sequential-pipeline-content-editorial-short-movie-builder.
Java package io.akka.samples.shortmovieagents. Akka 3.6.0. HTTP port 9732.

Components to wire (exactly):

- 1 AutonomousAgent MovieAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/movie-agent.md>) and four .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  SCRIPT, STORYBOARD, ASSEMBLE, and REVIEW tool sets are ALL registered on the agent; content
  safety gating is the job of ContentSafetyGuard, NOT of conditional .tools(...) wiring. The
  after-llm-response guardrail (ContentSafetyGuard) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow MovieProductionWorkflow per productionId with five steps:
  * scriptStep — emits ScriptingStarted on the entity, then calls componentClient
    .forAutonomousAgent(MovieAgent.class, "agent-" + productionId).runSingleTask(
      TaskDef.instructions("Brief: " + brief + "\nPhase: SCRIPT\nUse generateScenes and
      writeDialogueLine to produce a MovieScript.")
        .metadata("productionId", productionId)
        .metadata("phase", "SCRIPT")
        .taskType(MovieTasks.WRITE_SCRIPT)
    ). Reads forTask(taskId).result(WRITE_SCRIPT) to get MovieScript. Writes
    MovieEntity.recordScript(movieScript). WorkflowSettings.stepTimeout 60s.
  * storyboardStep — emits StoryboardingStarted, then runSingleTask with
    TaskDef.instructions(formatStoryboardContext(movieScript, brief)) and metadata.phase =
    "STORYBOARD", taskType DESIGN_STORYBOARD. Writes MovieEntity.recordStoryboard(storyboard).
    stepTimeout 60s.
  * assembleStep — emits AssemblingStarted, then runSingleTask with TaskDef.instructions
    (formatAssembleContext(storyboard, movieScript, brief)) and metadata.phase = "ASSEMBLE",
    taskType ASSEMBLE_PACKAGE. Writes MovieEntity.recordPackage(assembledPackage).
    stepTimeout 60s.
  * reviewStep — emits ReviewingStarted, then runSingleTask with TaskDef.instructions
    (formatReviewContext(assembledPackage, storyboard, movieScript)) and metadata.phase =
    "REVIEW", taskType REVIEW_PACKAGE. Runs PackageScorer over (assembledPackage, storyboard,
    movieScript) and writes MovieEntity.recordReview(reviewResult) then
    MovieEntity.recordCoherence(coherenceResult). stepTimeout 60s.
  * (error) step writes ProductionFailed and ends. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(MovieProductionWorkflow::error).

- 1 EventSourcedEntity MovieEntity (one per productionId). State MovieRecord{productionId,
  brief: Optional<String>, script: Optional<MovieScript>, storyboard: Optional<Storyboard>,
  assembledPackage: Optional<AssembledPackage>, reviewResult: Optional<ReviewResult>,
  coherenceResult: Optional<CoherenceResult>, status: ProductionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, safetyBlocks: List<SafetyBlock>}.
  ProductionStatus enum: CREATED, SCRIPTING, SCRIPTED, STORYBOARDING, STORYBOARDED, ASSEMBLING,
  ASSEMBLED, REVIEWING, REVIEWED, FAILED.
  Events: ProductionCreated{brief}, ScriptingStarted, ScriptWritten{script},
  StoryboardingStarted, StoryboardDesigned{storyboard}, AssemblingStarted,
  PackageAssembled{assembledPackage}, ReviewingStarted, ReviewCompleted{reviewResult},
  CoherenceScoredEvent{coherenceResult}, SafetyBlockRecorded{phase, violationType, reason,
  blockedAt}, ProductionFailed{reason}.
  Commands: create, startScripting, recordScript, startStoryboarding, recordStoryboard,
  startAssembling, recordPackage, startReviewing, recordReview, recordCoherence,
  recordSafetyBlock, fail, getProduction. emptyState() returns MovieRecord.initial("") with
  all Optional fields as Optional.empty() and no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside the
  event-applier.

- 1 View MovieView with row type MovieRow that mirrors MovieRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes MovieEntity events. ONE query
  getAllProductions: SELECT * AS productions FROM movie_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * MovieEndpoint at /api with POST /productions (body {brief}; mints productionId; calls
    MovieEntity.create(brief); then starts MovieProductionWorkflow with id
    "pipeline-" + productionId; returns {productionId}), GET /productions (list from
    getAllProductions, sorted newest-first), GET /productions/{id} (one row), GET
    /productions/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- MovieTasks.java declaring four Task<R> constants:
    WRITE_SCRIPT = Task.name("Write script").description("Generate a MovieScript with scenes
      and dialogue from the creative brief").resultConformsTo(MovieScript.class);
    DESIGN_STORYBOARD = Task.name("Design storyboard").description("Plan shots and framings
      for each scene in the MovieScript").resultConformsTo(Storyboard.class);
    ASSEMBLE_PACKAGE = Task.name("Assemble package").description("Build a renderable
      AssembledPackage by pairing shots with scenes").resultConformsTo(AssembledPackage.class);
    REVIEW_PACKAGE = Task.name("Review package").description("Check scene coherence and
      produce a ReviewResult summarising any gaps").resultConformsTo(ReviewResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {SCRIPT, STORYBOARD, ASSEMBLE, REVIEW}. Each function-tool method carries
  the constant phase in a parallel registry built at startup that ContentSafetyGuard reads to
  annotate safety-block events with the active phase.

- ScriptTools.java — @FunctionTool generateScenes(String brief) -> List<Scene> reading from
  src/main/resources/sample-data/briefs/*.json keyed by brief slug; @FunctionTool
  writeDialogueLine(String sceneDescription) -> String producing a dialogue line for the scene.

- StoryboardTools.java — @FunctionTool planShot(Scene scene) -> Shot (one Shot per Scene,
  with shotId minted as "s-" + sha1(scene.sceneId()).substring(0,8), framing assigned from a
  deterministic lookup table by SceneType); @FunctionTool selectFraming(Shot shot) -> String
  returning the preferred framing label for that shot.

- AssemblyTools.java — @FunctionTool buildPackageScene(Shot shot, Scene scene) -> PackageScene
  (assetPlaceholder constructed as "asset-" + shot.shotId() + "-" + scene.sceneId(),
  orderIndex from shot list position); @FunctionTool computeRuntime(List<Shot> shots) -> int
  (sum of shot.durationSeconds()).

- ReviewTools.java — @FunctionTool checkSceneCoherence(Scene scene, Shot shot) ->
  CoherenceCheck (coherent = true iff shot.sceneId() equals scene.sceneId()); @FunctionTool
  generateReviewSummary(List<CoherenceCheck> checks) -> String (lists any incoherent scenes
  by sceneId; "all scenes coherent" if none).

- ContentSafetyGuard.java — implements the after-llm-response hook. Receives the completed
  task's text payload, applies four rule checks (explicit/graphic content keyword scan,
  regulated-brand-name list lookup, legally-restricted-claim pattern match, incoherent-violence
  keyword scan), and either accepts or returns
  Guardrail.reject("safety-block: <violationType>: <reason>"). On reject ALSO calls
  MovieEntity.recordSafetyBlock(phase, violationType, reason) so the block is visible in the
  UI's safety-block log strip and in the audit log.

- PackageScorer.java — pure deterministic logic (no LLM). Inputs: AssembledPackage, Storyboard,
  MovieScript. Outputs: CoherenceResult with score and rationale. Four checks, one point per
  check satisfied, starting from a base of 1:
  (1) scene coverage — every Scene in MovieScript has a PackageScene;
  (2) shot reference validity — every PackageScene.shotId appears in Storyboard.shots;
  (3) runtime plausibility — totalRuntimeSeconds > 0 and > 0.5 * scenes.size() * 3;
  (4) order consistency — packageScenes are ordered by orderIndex ascending with no gaps.
  Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9732 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/briefs.jsonl with 5 seeded brief lines covering the three
  surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/briefs/*.json — three files keyed by seeded brief slug, each
  carrying 4-6 Scene entries with deterministic content so ScriptTools.generateScenes returns
  the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (H1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false (creative briefs
  are intent descriptions, not person-level data), decisions.authority_level = recommend-only
  (the movie package is a creative artefact, not an automated action), oversight.human_in_loop
  = true (a producer reviews the package before rendering), operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline, failure.failure_modes including
  "policy-violating-content", "missing-scene-coverage", "broken-shot-reference",
  "incoherent-package-order"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/movie-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Short Movie Builder", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of production cards; right = selected-production detail with brief header, script
  scenes table, storyboard shot list, assembled package scenes, coherence-score chip,
  safety-block log strip). Browser title exactly:
  <title>Akka Sample: Short Movie Builder</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per call
  (seedFor(productionId)), and deserialises into the task's typed return. Within a single task
  run, the mock replays the "tool_calls" array in order before returning the final typed result.
- Per-task mock-response formats for THIS blueprint:
    write-script.json — 5 MovieScript entries, each with 3-6 Scene items per seeded brief.
      Plus 1 deliberately POLICY-VIOLATING entry whose script text contains a banned keyword —
      ContentSafetyGuard rejects it, the mock then falls through to a clean script. The mock
      selects the violating entry on the FIRST iteration of every 3rd production (modulo seed)
      so J2 is reproducible.
    design-storyboard.json — 5 Storyboard entries paired one-to-one with the script entries,
      each with 3-6 Shot items.
    assemble-package.json — 5 AssembledPackage entries paired one-to-one. Plus 1 deliberately
      BROKEN-REFERENCE entry whose first PackageScene cites a shotId absent from the paired
      Storyboard — PackageScorer scores it 1; J3 verifies this.
    review-package.json — 5 ReviewResult entries paired one-to-one, each with CoherenceCheck
      items per scene.
- A MockModelProvider.seedFor(productionId) helper makes per-production selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. MovieAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion MovieTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (scriptStep
  60s, storyboardStep 60s, assembleStep 60s, reviewStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the MovieRecord row record is Optional<T>.
- Lesson 7: MovieTasks.java with WRITE_SCRIPT, DESIGN_STORYBOARD, ASSEMBLE_PACKAGE,
  REVIEW_PACKAGE constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9732 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements — no more.
- The single-agent invariant: exactly ONE AutonomousAgent (MovieAgent). PackageScorer is
  deterministic and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent; the
  after-llm-response guardrail (ContentSafetyGuard) is the runtime mechanism for content
  safety. Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: scriptStep writes MovieScript onto the
  entity, storyboardStep reads it and builds the STORYBOARD task's instruction context from it,
  assembleStep reads both, reviewStep reads all three. The agent itself is stateless across
  phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
