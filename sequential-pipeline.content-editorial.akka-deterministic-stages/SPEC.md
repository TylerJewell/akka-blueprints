# SPEC — deterministic-multi-stage-agent-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Deterministic Multi-Stage Agent Pipeline.
**One-line pitch:** A user submits a creative prompt; one `StoryAgent` walks it through three fixed stages — **OUTLINE** story beats, **WRITE** the body paragraphs, **WRITE** the ending — with each stage gated on the prior stage's validated output and each stage's output verified by an `after-llm-response` guardrail before the next stage starts.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain. One `StoryAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit stage dependency**: the OUTLINE stage's typed output becomes the WRITE_BODY stage's instruction context; the WRITE_BODY stage's typed output becomes the WRITE_ENDING stage's instruction context. The agent never holds all three stages in one conversation — each stage runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- An **`after-llm-response` guardrail** sits between the agent's raw output and the workflow's advancement step. After every stage task returns, `StageOutputGuardrail` validates the typed result: the OUTLINE result must carry at least two `Beat` entries with non-empty text; the BODY result must reference at least one beat id from the upstream `Outline`; the ENDING result must reference the same story title as the upstream `Outline` and contain a non-empty closing paragraph. On validation failure, the guardrail returns a structured `structural-violation` error to the agent loop so the task retries within its 4-iteration budget, and the workflow records a `GuardrailRejected{stage, field, reason}` event for visibility.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the stage-boundary handoffs are the right cut to enforce both the dependency contract and the per-stage output integrity.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **prompt** into the input (or picks one of three seeded prompts — `The lighthouse keeper's last night`, `A city that forgot the sun`, `The message that arrived too late`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/stories` and receives a `storyId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `OUTLINING` — the workflow has started `outlineStep` and the agent has been handed the OUTLINE_STORY task.
4. Within ~10–20 s the card reaches `BODY_WRITING` — the typed `Outline` is visible in the card detail (a list of beats with beat id and text). The agent's OUTLINE task returned; the workflow recorded `OutlineProduced` and ran the WRITE_BODY task.
5. Within ~10–20 s more the card reaches `ENDING_WRITING`. The `Body` is visible (a list of paragraph records with beat references).
6. Within ~10–20 s more the card reaches `VALIDATED`. The right pane now shows the full typed `Story` — title, genre, per-beat `Section`, and ending — plus a coherence score chip (1–5) and a one-line rationale.
7. The user can submit another prompt; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `StoryEndpoint` | `HttpEndpoint` | `/api/stories/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `StoryEntity`, `StoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `StoryEntity` | `EventSourcedEntity` | Per-story lifecycle: created → outlining → outlined → body-writing → body-written → ending-writing → ending-written → validated. Source of truth. | `StoryEndpoint`, `StoryPipelineWorkflow` | `StoryView` |
| `StoryPipelineWorkflow` | `Workflow` | One workflow per story. Steps: `outlineStep` → `bodyStep` → `endingStep` → `validateStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `StoryEndpoint` after `CREATED` | `StoryAgent`, `StoryEntity` |
| `StoryAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `StoryTasks.java`: `OUTLINE_STORY` → `Outline`, `WRITE_BODY` → `Body`, `WRITE_ENDING` → `Ending`. Each task is registered with the stage-appropriate function tools. | invoked by `StoryPipelineWorkflow` | returns typed results |
| `OutlineTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `generateBeats(prompt)` and `classifyGenre(prompt)`. Reads from `src/main/resources/sample-data/genres/*.json` for deterministic offline output. | called from OUTLINE task | returns `List<Beat>` / `String` |
| `BodyTools` | function-tools class | Implements `expandBeat(beat)` and `linkParagraphs(paragraphs)`. Pure in-memory transformations. | called from WRITE_BODY task | returns `Paragraph` / `List<Paragraph>` |
| `EndingTools` | function-tools class | Implements `resolveArcs(body)` and `composeClosure(arcs)`. | called from WRITE_ENDING task | returns `List<Arc>` / `String` |
| `StageOutputGuardrail` | `after-llm-response` guardrail (registered on `StoryAgent`) | Validates the typed result produced by each stage task before the workflow advances. Rejects structurally incomplete outputs. | every task result on every stage | accept / structured-reject |
| `StoryStructureValidator` | plain class (no Akka primitive) | Pure deterministic structural validator. Inputs: `Story`, `Body`, `Outline`. Output: `ValidationResult{score, rationale}`. | called from `validateStep` | returns score |
| `StoryView` | `View` | Read model: one row per story for the UI. | `StoryEntity` events | `StoryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Beat(String beatId, String text, int position) {}

record Outline(String title, String genre, List<Beat> beats, Instant outlinedAt) {}

record Paragraph(String paragraphId, String beatId, String text) {}

record Body(List<Paragraph> paragraphs, Instant writtenAt) {}

record Arc(String arcId, String label, List<String> paragraphIds) {}

record Ending(String closingText, List<Arc> arcs, Instant endingWrittenAt) {}

record Story(
    String title,
    String genre,
    Outline outline,
    Body body,
    Ending ending
) {}

record ValidationResult(
    int score,            // 1..5
    String rationale,
    Instant validatedAt
) {}

record GuardrailRejection(
    String stage,
    String field,
    String reason,
    Instant rejectedAt
) {}

record StoryRecord(
    String storyId,
    Optional<String> prompt,
    Optional<Outline> outline,
    Optional<Body> body,
    Optional<Ending> ending,
    Optional<ValidationResult> validation,
    StoryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<GuardrailRejection> guardrailRejections
) {}

enum StoryStatus {
    CREATED, OUTLINING, OUTLINED, BODY_WRITING, BODY_WRITTEN,
    ENDING_WRITING, ENDING_WRITTEN, VALIDATED, FAILED
}
```

Events on `StoryEntity`: `StoryCreated`, `OutlineStarted`, `OutlineProduced`, `BodyStarted`, `BodyWritten`, `EndingStarted`, `EndingWritten`, `StoryValidated`, `GuardrailRejected`, `StoryFailed`.

Every nullable lifecycle field on the `StoryRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/stories` — body `{ prompt }` → `{ storyId }`.
- `GET /api/stories` — list all stories, newest-first.
- `GET /api/stories/{id}` — one story.
- `GET /api/stories/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Deterministic Multi-Stage Agent Pipeline</title>`.

The App UI tab is a two-column layout: a left rail with the live list of stories (status pill + prompt + age) and a right pane with the selected story's detail — prompt, outline beats list, body paragraphs, ending text, validation score chip, and a guardrail-rejection log strip if any stage-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `after-llm-response` guardrail (stage-output validation)**: `StageOutputGuardrail` is registered on `StoryAgent` and runs after every task returns its typed result, before the workflow's advancement step executes. The validator applies per-stage structural rules: the OUTLINE result must carry `beats.size() >= 2`, each Beat with non-empty `text` and a unique `beatId`; the BODY result must have `paragraphs.size() >= beats.size()` and each `Paragraph.beatId` must appear in the upstream `Outline.beats`; the ENDING result must have `closingText.length() > 0` and `arcs.size() >= 1`. On rejection, the guardrail returns a structured `structural-violation` error naming the failing field, and the workflow records a `GuardrailRejected{stage, field, reason}` event. The agent loop retries within its 4-iteration budget.
- **E1 — `on-decision-eval`**: runs immediately after `EndingWritten` lands, as `validateStep` inside the workflow. `StoryStructureValidator` is a deterministic rule-based scorer (no LLM call — the eval is rule-based on purpose, so the same story always scores the same): the outline must carry a non-empty `genre` (genre detection), every `Paragraph.beatId` must reference a beat in the `Outline` (beat coverage), the `Ending.arcs` must reference at least one `Paragraph.paragraphId` from the `Body` (arc linkage), and the story title must match `Outline.title` (title consistency). Emits `StoryValidated{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `StoryAgent` → `prompts/story-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that stage's tool set, treat each task's typed input as the entire context for that stage, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded prompt `The lighthouse keeper's last night`; within 60 s the story reaches `VALIDATED` with ≥ 2 beats, ≥ 2 paragraphs, a non-empty ending, and a validation score chip on the card.
2. **J2** — The agent's first iteration on a story returns an OUTLINE with only one beat (mock LLM path). `StageOutputGuardrail` rejects the result; a `GuardrailRejected` event lands on the entity; the agent retries and returns a valid Outline; the story eventually completes correctly. The UI's rejection-log strip shows the one rejected result.
3. **J3** — A story whose mock-LLM trajectory produces a Paragraph citing a beatId absent from the recorded Outline is rejected by the guardrail; the UI flags the card in the rejection-log strip.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-story trace (logged at `INFO`); the OUTLINE task's log shows only OUTLINE-tool calls, the WRITE_BODY task's log shows only BODY-tool calls, the WRITE_ENDING task's log shows only ENDING-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named deterministic-multi-stage-agent-pipeline demonstrating the
sequential-pipeline x content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact sequential-pipeline-content-editorial-akka-deterministic-stages.
Java package io.akka.samples.deterministicmultistageagentpipeline. Akka 3.6.0. HTTP port 9724.

Components to wire (exactly):

- 1 AutonomousAgent StoryAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/story-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  OUTLINE, BODY, and ENDING tool sets are ALL registered on the agent; stage-output gating is
  the job of StageOutputGuardrail, NOT of conditional .tools(...) wiring. The after-llm-response
  guardrail (StageOutputGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow StoryPipelineWorkflow per storyId with four steps:
  * outlineStep — emits OutlineStarted on the entity, then calls componentClient
    .forAutonomousAgent(StoryAgent.class, "agent-" + storyId).runSingleTask(
      TaskDef.instructions("Prompt: " + prompt + "\nStage: OUTLINE\nUse generateBeats and
      classifyGenre to build a structured story outline for this prompt.")
        .metadata("storyId", storyId)
        .metadata("stage", "OUTLINE")
        .taskType(StoryTasks.OUTLINE_STORY)
    ). Reads forTask(taskId).result(OUTLINE_STORY) to get Outline. Writes
    StoryEntity.recordOutline(outline). WorkflowSettings.stepTimeout 60s.
  * bodyStep — emits BodyStarted, then runSingleTask with TaskDef.instructions
    (formatBodyContext(outline, prompt)) and metadata.stage = "WRITE_BODY", taskType
    WRITE_BODY. Writes StoryEntity.recordBody(body). stepTimeout 60s.
  * endingStep — emits EndingStarted, then runSingleTask with TaskDef.instructions
    (formatEndingContext(body, outline, prompt)) and metadata.stage = "WRITE_ENDING", taskType
    WRITE_ENDING. Writes StoryEntity.recordEnding(ending). stepTimeout 60s.
  * validateStep — runs the deterministic StoryStructureValidator over (outline, body,
    ending) and writes StoryEntity.recordValidation(validation). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(StoryPipelineWorkflow::error). The error step writes
  StoryFailed and ends.

- 1 EventSourcedEntity StoryEntity (one per storyId). State StoryRecord{storyId,
  prompt: Optional<String>, outline: Optional<Outline>, body: Optional<Body>,
  ending: Optional<Ending>, validation: Optional<ValidationResult>, status: StoryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>, guardrailRejections: List<GuardrailRejection>}.
  StoryStatus enum: CREATED, OUTLINING, OUTLINED, BODY_WRITING, BODY_WRITTEN,
  ENDING_WRITING, ENDING_WRITTEN, VALIDATED, FAILED. Events:
  StoryCreated{prompt}, OutlineStarted, OutlineProduced{outline}, BodyStarted,
  BodyWritten{body}, EndingStarted, EndingWritten{ending},
  StoryValidated{validation}, GuardrailRejected{stage, field, reason, rejectedAt}, StoryFailed{reason}.
  Commands: create, startOutline, recordOutline, startBody, recordBody, startEnding,
  recordEnding, recordValidation, recordGuardrailRejection, fail, getStory. emptyState()
  returns StoryRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View StoryView with row type StoryRow that mirrors StoryRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes StoryEntity events. ONE
  query getAllStories: SELECT * AS stories FROM story_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * StoryEndpoint at /api with POST /stories (body {prompt}; mints storyId; calls
    StoryEntity.create(prompt); then starts StoryPipelineWorkflow with id
    "pipeline-" + storyId; returns {storyId}), GET /stories (list from
    getAllStories, sorted newest-first), GET /stories/{id} (one row), GET
    /stories/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- StoryTasks.java declaring three Task<R> constants:
    OUTLINE_STORY = Task.name("Outline story").description("Build a structured beat outline
      for the prompt by calling generateBeats and classifyGenre").resultConformsTo(Outline.class);
    WRITE_BODY = Task.name("Write body").description("Expand each outline beat into a paragraph
      by calling expandBeat and linkParagraphs").resultConformsTo(Body.class);
    WRITE_ENDING = Task.name("Write ending").description("Resolve narrative arcs and compose a
      closing paragraph by calling resolveArcs and composeClosure").resultConformsTo(Ending.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Stage.java — enum {OUTLINE, WRITE_BODY, WRITE_ENDING}. Each function-tool method is
  annotated with the constant stage, e.g. @FunctionTool(name = "generateBeats", stage =
  Stage.OUTLINE) (use a custom annotation if the SDK's @FunctionTool does not carry a stage
  field — the guardrail reads it from a parallel registry built at startup if so).

- OutlineTools.java — @FunctionTool generateBeats(String prompt) -> List<Beat> reading from
  src/main/resources/sample-data/genres/*.json keyed by prompt slug; @FunctionTool
  classifyGenre(String prompt) -> String reading the genre field from the matching entry.

- BodyTools.java — @FunctionTool expandBeat(Beat beat) -> Paragraph (paragraphId minted as
  "p-" + sha1(beat.beatId()).substring(0,8), text is a 2-sentence expansion of beat.text());
  @FunctionTool linkParagraphs(List<Paragraph> paragraphs) -> List<Paragraph> (deterministic
  ordering — sort by beat position, prepend a linking phrase to each paragraph body after
  the first).

- EndingTools.java — @FunctionTool resolveArcs(Body body) -> List<Arc> (one Arc per
  distinct thematic thread inferred from paragraph texts — deterministic clustering by first
  noun phrase); @FunctionTool composeClosure(List<Arc> arcs) -> String (a closing paragraph
  weaving the arc labels).

- StageOutputGuardrail.java — implements the after-llm-response hook. Reads the task's
  declared stage from TaskDef metadata, retrieves the typed result from the agent's response,
  and applies per-stage structural rules:
    OUTLINE: beats.size() >= 2, all beatIds unique, all beat texts non-empty.
    WRITE_BODY: paragraphs.size() >= upstream outline beats.size(), every Paragraph.beatId
      appears in outline.beats[].beatId().
    WRITE_ENDING: closingText non-empty, arcs.size() >= 1, every Arc.paragraphIds[i]
      appears in body.paragraphs[].paragraphId().
  On rejection, returns Guardrail.reject("structural-violation: <field> <constraint>, saw
  <actual>") AND calls StoryEntity.recordGuardrailRejection(stage, field, reason) so the
  rejection is visible in the UI's rejection-log strip and in the audit log.

- StoryStructureValidator.java — pure deterministic logic (no LLM). Inputs: Outline, Body,
  Ending. Outputs: ValidationResult with score and rationale. Four checks, one point per
  check satisfied, starting from a base of 1: genre detection (outline.genre() non-empty),
  beat coverage (every outline beat referenced by at least one Paragraph.beatId), arc linkage
  (every Arc.paragraphIds[0] references a Paragraph.paragraphId in the body), and title
  consistency (story title matches outline.title()). Score range 1-5. Rationale names the
  largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9724 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/prompts.jsonl with 5 seeded prompt lines covering the
  three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/genres/*.json — three files keyed by seeded prompt slug,
  each carrying a genre classification and 4-6 Beat entries with deterministic content so
  OutlineTools.generateBeats returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (prompts are
  creative, not person-level), decisions.authority_level = recommend-only (the story is
  creative output), oversight.human_in_loop = true (a human reviews the story before
  publishing), operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "missing-beats", "orphaned-paragraph", "arc-linkage-failure",
  "title-mismatch", "structural-violation"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/story-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Deterministic Multi-Stage Agent
  Pipeline", prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of story cards; right = selected-story detail with prompt header, outline beats
  list, body paragraphs, ending text, validation-score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Deterministic Multi-Stage Agent Pipeline</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(storyId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    outline-story.json — 6 Outline entries, each with 3-5 Beat items and a genre string.
      Each entry's tool_calls array contains 2 calls: 1 classifyGenre(prompt) + 1
      generateBeats(prompt). Plus 1 deliberately INCOMPLETE entry whose Outline carries
      only one Beat — the guardrail rejects it, the mock then falls through to a normal
      outline sequence. The mock should select the violating entry on the FIRST iteration
      of every 3rd story (modulo seed) so J2 is reproducible.
    write-body.json — 6 Body entries paired one-to-one with the outline entries, each with
      3-5 Paragraph items referencing valid beatIds from the paired Outline, with tool_calls
      containing expandBeat (one per beat) + linkParagraphs in order. Plus 1 entry where one
      Paragraph.beatId does not match any beat in the paired Outline — the guardrail rejects
      this; J3 verifies.
    write-ending.json — 6 Ending entries paired one-to-one. Each carries a non-empty
      closingText, 1-3 Arc items referencing paragraphIds from the paired Body, tool_calls
      containing resolveArcs + composeClosure.
- A MockModelProvider.seedFor(storyId) helper makes per-story selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. StoryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion StoryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (outlineStep
  60s, bodyStep 60s, endingStep 60s, validateStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the StoryRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: StoryTasks.java with OUTLINE_STORY, WRITE_BODY, WRITE_ENDING constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9724 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (StoryAgent). The
  on-decision eval is rule-based (StoryStructureValidator.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each stage's tool set is registered on the agent, but
  the after-llm-response guardrail (StageOutputGuardrail) is the runtime mechanism that
  validates stage outputs before the workflow advances. Do NOT conditionally register tools
  per task — the guardrail is the gate.
- Task dependency is carried by typed task results: outlineStep writes Outline onto the
  entity, bodyStep reads it and builds the WRITE_BODY task's instruction context from it,
  endingStep reads both. The agent itself is stateless across stages.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
