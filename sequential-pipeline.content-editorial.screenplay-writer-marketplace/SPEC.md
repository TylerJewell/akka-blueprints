# SPEC — screenplay-writer-marketplace

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Screenplay Writer Team.
**One-line pitch:** A user submits text or an email thread; one `ScreenplayAgent` walks it through three task phases — **PARSE** source material into structured story elements, **DEVELOP** those elements into a scene plan, **FORMAT** the scene plan into a production-ready screenplay — with PII stripped from emails before the agent sees them and a final delivery check that blocks output containing residual PII.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain. One `ScreenplayAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PARSE task's typed output becomes the DEVELOP task's instruction context; the DEVELOP task's typed output becomes the FORMAT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and every final response delivery. After the FORMAT task completes and the workflow is about to mark the screenplay as delivered, `DeliveryGuardrail` scans the formatted screenplay text for PII patterns (email addresses, phone numbers, full names matching a blocklist derived from the source email headers). If any PII token is found, the guardrail blocks delivery, records a `DeliveryBlocked` event on the entity, and returns a structured error. The workflow's `deliverStep` then routes to an error step that transitions the entity to `DELIVERY_BLOCKED`. The guardrail is the last line of defence — it runs after all three pipeline phases have completed.
- A **`pii` sanitizer** runs immediately after source material is received by `ScreenplayEndpoint`, before any data is written to `ScreenplayEntity`. The sanitizer is a deterministic, rule-based in-process component (no LLM call). It identifies PII tokens in the submitted text and email headers using a pattern registry (email-address regex, phone-number regex, name patterns extracted from `From:` / `To:` headers) and replaces each token with a stable placeholder (`[EMAIL_1]`, `[PHONE_1]`, `[NAME_1]`) so the agent works on sanitized material throughout. The original-to-placeholder mapping is stored privately on the entity and is never surfaced in any user-facing response.

The blueprint shows that a sequential content pipeline can run cleanly on sensitive source material without exposing PII to the LLM — the sanitizer and the delivery guardrail act at the two natural boundaries (ingestion and egress) rather than asking the agent to make PII decisions in the middle of a creative task.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes **source material** into the input (or picks one of three seeded samples — `Merger announcement email thread`, `Product launch campaign emails`, `Investor update email set`).
2. The user clicks **Write screenplay**. The UI POSTs to `/api/screenplays` and receives a `screenplayId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PARSING` — the workflow has started `parseStep` and the PII sanitizer has run; the agent has been handed the PARSE task on the sanitized material.
4. Within ~10–20 s the card reaches `DEVELOPING` — the typed `ParsedSource` is visible in the card detail (a table of story elements: characters, settings, dramatic beats). The agent's PARSE task returned; the workflow recorded `SourceParsed` and ran the DEVELOP task.
5. Within ~10–20 s more the card reaches `FORMATTING`. The `ScenePlan` is visible (scene list with sluglines and beat assignments).
6. Within ~10–20 s more the card reaches `DELIVERED`. The right pane now shows the full typed `Screenplay` — title, logline, per-scene `SceneBlock` with action lines and dialogue — plus a delivery-check result chip (PASSED or BLOCKED).
7. The user can submit another source text; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ScreenplayEndpoint` | `HttpEndpoint` | `/api/screenplays/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ScreenplayEntity`, `ScreenplayView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ScreenplayEntity` | `EventSourcedEntity` | Per-screenplay lifecycle: created → parsing → parsed → developing → developed → formatting → formatted → delivered / delivery-blocked. Source of truth. Holds private placeholder-map. | `ScreenplayEndpoint`, `ScreenplayPipelineWorkflow` | `ScreenplayView` |
| `ScreenplayPipelineWorkflow` | `Workflow` | One workflow per screenplay. Steps: `parseStep` → `developStep` → `formatStep` → `deliverStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `ScreenplayEndpoint` after `CREATED` | `ScreenplayAgent`, `ScreenplayEntity` |
| `ScreenplayAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `ScreenplayTasks.java`: `PARSE_SOURCE` → `ParsedSource`, `DEVELOP_SCENES` → `ScenePlan`, `FORMAT_SCREENPLAY` → `Screenplay`. Each task is registered with the phase-appropriate function tools. The `before-agent-response` guardrail (`DeliveryGuardrail`) is registered on the agent. | invoked by `ScreenplayPipelineWorkflow` | returns typed results |
| `ParseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `extractCharacters(text)`, `extractSettings(text)`, and `extractBeats(text)`. Reads sanitized source text from entity state. | called from PARSE task | returns `List<Character>`, `List<Setting>`, `List<Beat>` |
| `DevelopTools` | function-tools class | Implements `buildSceneOutline(characters, settings, beats)` and `assignBeatsToScenes(outline, beats)`. Pure in-memory transformations. | called from DEVELOP task | returns `List<SceneOutline>` |
| `FormatTools` | function-tools class | Implements `formatSlugline(scene)`, `formatAction(scene)`, and `formatDialogue(scene, characters)`. | called from FORMAT task | returns `SceneBlock` |
| `PiiSanitizer` | plain class (no Akka primitive) | Deterministic rule-based in-process sanitizer. Runs at ingestion inside `ScreenplayEndpoint` before any entity write. Identifies PII tokens (email addresses, phone numbers, names from headers) and replaces them with stable numbered placeholders. Returns `SanitizedSource` and a `PlaceholderMap`. | called from `ScreenplayEndpoint` | returns `SanitizedSource`, `PlaceholderMap` |
| `DeliveryGuardrail` | `before-agent-response` guardrail (registered on `ScreenplayAgent`) | Scans the agent's final formatted screenplay text for PII patterns. On detection, blocks delivery and returns a structured `pii-detected` error. On pass, allows the response through. | every final response from FORMAT task | accept / structured-reject |
| `ScreenplayView` | `View` | Read model: one row per screenplay for the UI. | `ScreenplayEntity` events | `ScreenplayEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Character(String characterId, String placeholder, String archetype) {}

record Setting(String settingId, String placeholder, String type) {}

record Beat(String beatId, String description, String dramatic_function) {}

record ParsedSource(
    List<Character> characters,
    List<Setting> settings,
    List<Beat> beats,
    Instant parsedAt
) {}

record SceneOutline(
    String sceneId,
    String slugline,
    List<String> beatIds,
    List<String> characterIds,
    String settingId
) {}

record ScenePlan(List<SceneOutline> scenes, Instant developedAt) {}

record DialogueLine(String characterId, String line) {}

record SceneBlock(
    String sceneId,
    String slugline,
    String action,
    List<DialogueLine> dialogue
) {}

record Screenplay(
    String title,
    String logline,
    List<SceneBlock> scenes,
    Instant formattedAt
) {}

record PlaceholderMap(Map<String, String> originalToPlaceholder) {}

record SanitizedSource(String sanitizedText, PlaceholderMap placeholderMap) {}

record DeliveryCheckResult(
    boolean passed,
    List<String> detectedTokens,
    Instant checkedAt
) {}

record ScreenplayRecord(
    String screenplayId,
    Optional<String> sourceTitle,
    Optional<ParsedSource> parsedSource,
    Optional<ScenePlan> scenePlan,
    Optional<Screenplay> screenplay,
    Optional<DeliveryCheckResult> deliveryCheck,
    ScreenplayStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ScreenplayStatus {
    CREATED, PARSING, PARSED, DEVELOPING, DEVELOPED,
    FORMATTING, FORMATTED, DELIVERED, DELIVERY_BLOCKED, FAILED
}
```

Events on `ScreenplayEntity`: `ScreenplayCreated`, `ParseStarted`, `SourceParsed`, `DevelopStarted`, `ScenesDevoped`, `FormatStarted`, `ScreenplayFormatted`, `ScreenplayDelivered`, `DeliveryBlocked`, `ScreenplayFailed`.

Every nullable lifecycle field on the `ScreenplayRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/screenplays` — body `{ sourceTitle, sourceText }` → `{ screenplayId }`.
- `GET /api/screenplays` — list all screenplays, newest-first.
- `GET /api/screenplays/{id}` — one screenplay.
- `GET /api/screenplays/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Screenplay Writer Team</title>`.

The App UI tab is a two-column layout: a left rail with the live list of screenplays (status pill + source title + age) and a right pane with the selected screenplay's detail — source title, extracted characters/settings/beats table, scene plan outline, formatted screenplay scenes, delivery-check chip, and a delivery-block log strip if the guardrail fired.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (delivery gate)**: `DeliveryGuardrail` is registered on `ScreenplayAgent` and runs before the agent's final response in the FORMAT task is returned to the workflow. It scans the formatted screenplay text using the same pattern registry as `PiiSanitizer` — email-address regex, phone-number regex, and names from the source email headers (carried in the placeholder map stored on the entity). If any raw PII token is detected, the guardrail returns a structured `pii-detected` error to the workflow, the workflow records `DeliveryBlocked{detectedTokens, reason}` on the entity, and the entity transitions to `DELIVERY_BLOCKED`. The author must re-examine the source material and resubmit. If no PII is detected, the guardrail passes and the workflow records `ScreenplayDelivered`.
- **S1 — `pii` sanitizer (ingestion gate)**: `PiiSanitizer` runs inside `ScreenplayEndpoint.submitScreenplay()` before any write to `ScreenplayEntity`. It parses the submitted `sourceText` and email headers for PII tokens. Each unique token is assigned a stable numbered placeholder (`[EMAIL_1]`, `[PHONE_1]`, `[NAME_1]`). The sanitized text is stored on the entity's `SanitizedSource` field; the placeholder map is stored privately and never returned to the UI or passed to the agent. When `DeliveryGuardrail` runs its checks, it uses the placeholder map's keys (the original PII values) to verify they do not appear in the formatted output. This means the sanitizer and the guardrail share the same source-of-truth token list without ever exposing it.

## 9. Agent prompts

- `ScreenplayAgent` → `prompts/screenplay-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. It is told explicitly that all names, emails, and phone numbers in the source material have already been replaced by placeholders — it must use the placeholders as-is and never attempt to recover or infer the originals.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded source `Merger announcement email thread`; within 60 s the screenplay reaches `DELIVERED` with non-empty characters, ≥ 2 scenes, and a delivery-check chip showing PASSED.
2. **J2** — An email containing a raw email address and a phone number is submitted. The `PiiSanitizer` replaces them with `[EMAIL_1]` and `[PHONE_1]` before the entity write. The formatted screenplay output contains only the placeholders; no raw PII appears anywhere in the record.
3. **J3** — A mock FORMAT task response that includes a raw email address (bypassing the sanitizer) is blocked by `DeliveryGuardrail`; a `DeliveryBlocked` event lands on the entity; the UI's delivery-block strip shows the detected token.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-screenplay trace (logged at `INFO`); the PARSE task's log shows only PARSE-tool calls, the DEVELOP task's log shows only DEVELOP-tool calls, the FORMAT task's log shows only FORMAT-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sequential-pipeline-content-editorial-screenplay-writer-marketplace
demonstrating the sequential-pipeline x content-editorial cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-content-editorial-screenplay-writer-marketplace. Java package
io.akka.samples.screenplaywriterteam. Akka 3.6.0. HTTP port 9817.

Components to wire (exactly):

- 1 AutonomousAgent ScreenplayAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/screenplay-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the PARSE, DEVELOP, and FORMAT tool sets are ALL registered on the
  agent. The before-agent-response guardrail (DeliveryGuardrail) is registered on the agent
  via the agent's guardrail-configuration block. On guardrail rejection the workflow routes
  to the error step.

- 1 Workflow ScreenplayPipelineWorkflow per screenplayId with four steps:
  * parseStep — emits ParseStarted on the entity, then calls componentClient
    .forAutonomousAgent(ScreenplayAgent.class, "agent-" + screenplayId).runSingleTask(
      TaskDef.instructions("Source title: " + sourceTitle + "\nPhase: PARSE\nAnalyze the
      sanitized source text and extract characters, settings, and dramatic beats.")
        .metadata("screenplayId", screenplayId)
        .metadata("phase", "PARSE")
        .taskType(ScreenplayTasks.PARSE_SOURCE)
    ). Reads forTask(taskId).result(PARSE_SOURCE) to get ParsedSource. Writes
    ScreenplayEntity.recordParsedSource(parsedSource). WorkflowSettings.stepTimeout 60s.
  * developStep — emits DevelopStarted, then runSingleTask with TaskDef.instructions
    (formatDevelopContext(parsedSource, sourceTitle)) and metadata.phase = "DEVELOP", taskType
    DEVELOP_SCENES. Writes ScreenplayEntity.recordScenePlan(scenePlan). stepTimeout 60s.
  * formatStep — emits FormatStarted, then runSingleTask with TaskDef.instructions
    (formatFormatContext(scenePlan, parsedSource, sourceTitle)) and metadata.phase = "FORMAT",
    taskType FORMAT_SCREENPLAY. The before-agent-response guardrail (DeliveryGuardrail) runs
    against the agent's response before it is returned to the workflow. Writes
    ScreenplayEntity.recordScreenplay(screenplay) on pass. stepTimeout 60s.
  * deliverStep — runs delivery confirmation: if the entity status is FORMATTED (guardrail
    passed), writes ScreenplayDelivered. If DELIVERY_BLOCKED, writes the block and ends.
    stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(ScreenplayPipelineWorkflow::error). The error step writes
  ScreenplayFailed and ends.

- 1 EventSourcedEntity ScreenplayEntity (one per screenplayId). State ScreenplayRecord{
  screenplayId, sourceTitle: Optional<String>, parsedSource: Optional<ParsedSource>,
  scenePlan: Optional<ScenePlan>, screenplay: Optional<Screenplay>,
  deliveryCheck: Optional<DeliveryCheckResult>, status: ScreenplayStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. ScreenplayStatus enum: CREATED, PARSING, PARSED,
  DEVELOPING, DEVELOPED, FORMATTING, FORMATTED, DELIVERED, DELIVERY_BLOCKED, FAILED. Events:
  ScreenplayCreated{sourceTitle}, ParseStarted, SourceParsed{parsedSource}, DevelopStarted,
  ScenesDevoped{scenePlan}, FormatStarted, ScreenplayFormatted{screenplay},
  ScreenplayDelivered{deliveryCheck}, DeliveryBlocked{detectedTokens, reason},
  ScreenplayFailed{reason}. Commands: create, startParse, recordParsedSource, startDevelop,
  recordScenePlan, startFormat, recordScreenplay, recordDelivery, blockDelivery, fail,
  getScreenplay. emptyState() returns ScreenplayRecord.initial("") with all Optional fields
  as Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View ScreenplayView with row type ScreenplayRow that mirrors ScreenplayRecord exactly
  (all Optional<T> lifecycle fields preserved). Table updater consumes ScreenplayEntity
  events. ONE query getAllScreenplays: SELECT * AS screenplays FROM screenplay_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ScreenplayEndpoint at /api with POST /screenplays (body {sourceTitle, sourceText}; runs
    PiiSanitizer on sourceText; mints screenplayId; calls ScreenplayEntity.create(sourceTitle,
    sanitizedSource); then starts ScreenplayPipelineWorkflow with id
    "pipeline-" + screenplayId; returns {screenplayId}), GET /screenplays (list from
    getAllScreenplays, sorted newest-first), GET /screenplays/{id} (one row), GET
    /screenplays/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- ScreenplayTasks.java declaring three Task<R> constants:
    PARSE_SOURCE = Task.name("Parse source").description("Extract characters, settings, and
      beats from sanitized source text").resultConformsTo(ParsedSource.class);
    DEVELOP_SCENES = Task.name("Develop scenes").description("Build a scene outline and
      assign beats to scenes").resultConformsTo(ScenePlan.class);
    FORMAT_SCREENPLAY = Task.name("Format screenplay").description("Produce production-ready
      scene blocks with sluglines, action, and dialogue").resultConformsTo(Screenplay.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- PiiSanitizer.java — pure in-process class (no Akka primitive). Called from
  ScreenplayEndpoint.submitScreenplay(). Pattern registry covers: RFC-5322 email-address
  regex, E.164-format phone-number regex, display names extracted from "From:" and "To:"
  headers in the sourceText (format "Name <email@...>"). Assigns stable numbered placeholders
  per unique token. Returns SanitizedSource{sanitizedText, placeholderMap}.

- DeliveryGuardrail.java — implements the before-agent-response hook. Reads the PlaceholderMap
  (the map of original PII values) from the in-flight ScreenplayEntity state keyed by
  screenplayId (carried in the TaskDef metadata). For each original PII value in the map,
  checks whether it appears in the agent's formatted screenplay text. If any match: returns
  Guardrail.reject("pii-detected: <token> found in formatted output") and calls
  ScreenplayEntity.blockDelivery(detectedTokens, reason) for audit. On pass: allows the
  response through.

- ParseTools.java — @FunctionTool extractCharacters(String sanitizedText) -> List<Character>
  reading from the sanitized source and identifying speaker-name placeholders; @FunctionTool
  extractSettings(String sanitizedText) -> List<Setting> identifying location references;
  @FunctionTool extractBeats(String sanitizedText) -> List<Beat> identifying dramatic turning
  points.

- DevelopTools.java — @FunctionTool buildSceneOutline(List<Character>, List<Setting>,
  List<Beat>) -> List<SceneOutline> (assigns each beat to a scene with an appropriate
  slugline); @FunctionTool assignBeatsToScenes(List<SceneOutline>, List<Beat>) ->
  List<SceneOutline> (deterministic refinement pass — reassigns beats for pacing).

- FormatTools.java — @FunctionTool formatSlugline(SceneOutline) -> String (INT./EXT.
  uppercase slugline from the scene's settingId); @FunctionTool formatAction(SceneOutline,
  List<Beat>) -> String (present-tense action paragraph); @FunctionTool
  formatDialogue(SceneOutline, List<Character>) -> List<DialogueLine> (one line per
  character's contribution to this scene's beats).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9817 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/emails.jsonl with 5 seeded source entries covering the
  three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/sources/*.json — three files keyed by seeded source title,
  each carrying sanitized source text and a matching parsed-source fixture so ParseTools
  returns consistent output across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root with data.data_classes.pii = true (source emails may
  contain PII), decisions.authority_level = recommend-only (the screenplay is a creative
  artifact, not an autonomous action), oversight.human_in_loop = true (a human author reviews
  the screenplay before production), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "pii-in-output", "missing-scene-coverage",
  "delivery-blocked", "evidence-thin-scene"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/screenplay-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Screenplay Writer Team", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of screenplay cards; right = selected-screenplay detail with source title, parsed
  characters/settings/beats table, scene plan outline, formatted screenplay scenes, delivery-
  check chip, delivery-block log strip). Browser title exactly:
  <title>Akka Sample: Screenplay Writer Team</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json.
- Per-task mock-response shapes:
    parse-source.json — 5 ParsedSource entries with 2-4 Characters, 2-3 Settings, 4-6
      Beats each. Plus 1 entry where the parsed Characters list contains a placeholder name
      that mimics a raw email address — this triggers the DeliveryGuardrail on the final
      formatted output (J3).
    develop-scenes.json — 5 ScenePlan entries paired one-to-one, each with 3-5
      SceneOutline items and beat assignments.
    format-screenplay.json — 5 Screenplay entries paired one-to-one. Plus 1 entry whose
      action text in the first SceneBlock contains a raw email address (bypassing the mock's
      sanitizer path) — DeliveryGuardrail blocks this entry (J3).
- MockModelProvider.seedFor(screenplayId) makes per-screenplay selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ScreenplayAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ScreenplayTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (parseStep
  60s, developStep 60s, formatStep 60s, deliverStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the ScreenplayRecord row record is Optional<T>.
- Lesson 7: ScreenplayTasks.java with PARSE_SOURCE, DEVELOP_SCENES, FORMAT_SCREENPLAY
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9817 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ScreenplayAgent).
  PiiSanitizer and DeliveryGuardrail are deterministic in-process components; neither makes
  an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent; the
  pipeline order is enforced by task-chaining through typed results, not by conditionally
  registering tools.
- Task dependency is carried by typed task results: parseStep writes ParsedSource onto the
  entity, developStep reads it and builds the DEVELOP task's instruction context from it,
  formatStep reads both.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
