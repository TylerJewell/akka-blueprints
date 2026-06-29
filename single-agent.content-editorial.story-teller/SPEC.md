# SPEC — story-teller

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** StoryTeller.
**One-line pitch:** A user submits a creative prompt and optional style constraints; one AI agent reads the enriched prompt (passed as a task attachment, never as inline prompt text) and returns a fully-formed short story — title, narrative body, author's note — packaged as a typed `GeneratedStory`.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the content-editorial domain. One `StoryTellerAgent` (AutonomousAgent) carries the entire creative decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **content-safety filter** runs inside a Consumer between the raw prompt submission and the agent call — so the model never receives prompts that contain disallowed content categories. The filter tags genres, tone markers, and content flags; flagged prompts transition to `BLOCKED` without invoking the agent at all.
- A **before-agent-response guardrail** validates the agent's story on every turn: well-formed JSON, non-empty `title`, non-empty `body` (at least 150 characters), `genre` is in the allowed set, and `authorNote` is present. A malformed story triggers a retry inside the same task.
- An **on-decision-eval** runs immediately after each `StoryRecorded` event, scoring the story on quality dimensions (does the body match the requested genre? is the title consistent with the narrative? is the author's note informative?).

The blueprint shows that the single-agent pattern does not mean "ungoverned" — independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **genre** from a dropdown (Fantasy / Mystery / Sci-Fi / Romance / Horror / Custom) and types or pastes a **prompt** into the textarea (or picks one of three seeded examples — a fantasy quest seed, a mystery investigation seed, a sci-fi first-contact seed).
2. The user optionally sets **style constraints**: `tone` (whimsical / gritty / literary / neutral), `wordCountTarget` (100 / 300 / 600), and `pointOfView` (first / third).
3. The user clicks **Generate story**. The UI POSTs to `/api/stories` and receives a `storyId`.
4. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `ENRICHED` (safety check passed, metadata tags applied) or `BLOCKED` (safety check failed, reason displayed).
5. For enriched stories, within ~10–30 s the `generationStep` completes. The card transitions to `GENERATING` then `STORY_RECORDED`. The story appears: a title, the narrative body, and the author's note explaining genre and style choices.
6. Within ~1 s of the story, the `qualityStep` finishes. The card shows a **quality score** chip (1–5) plus a one-line rationale.
7. The user can submit another prompt; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `StoryEndpoint` | `HttpEndpoint` | `/api/stories/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `StoryEntity`, `StoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `StoryEntity` | `EventSourcedEntity` | Per-story lifecycle: requested → enriched/blocked → generating → recorded → scored. Source of truth. | `StoryEndpoint`, `PromptEnricher`, `StoryWorkflow` | `StoryView` |
| `PromptEnricher` | `Consumer` | Subscribes to `StoryRequested` events; applies content-safety check; tags genre, tone, flags; calls `StoryEntity.attachEnriched` or `StoryEntity.block`. | `StoryEntity` events | `StoryEntity` |
| `StoryWorkflow` | `Workflow` | One workflow per story. Steps: `awaitEnrichedStep` → `generationStep` → `qualityStep`. | started by `PromptEnricher` once enriched event lands | `StoryTellerAgent`, `StoryEntity` |
| `StoryTellerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives style constraints in the task definition and the enriched prompt as a task attachment; returns `GeneratedStory`. | invoked by `StoryWorkflow` | returns story |
| `StoryView` | `View` | Read model: one row per story for the UI. | `StoryEntity` events | `StoryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record StyleConstraints(
    Tone tone,
    int wordCountTarget,
    PointOfView pointOfView
) {}
enum Tone { WHIMSICAL, GRITTY, LITERARY, NEUTRAL }
enum PointOfView { FIRST, THIRD }

record StoryRequest(
    String storyId,
    String genre,
    String rawPrompt,
    StyleConstraints constraints,
    String requestedBy,
    Instant requestedAt
) {}

record EnrichedPrompt(
    String safePrompt,
    List<String> contentTags,
    boolean safetyPassed,
    String safetyReason
) {}

record GeneratedStory(
    String title,
    String body,
    String authorNote,
    String genre,
    Instant generatedAt
) {}

record QualityResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Story(
    String storyId,
    Optional<StoryRequest> request,
    Optional<EnrichedPrompt> enriched,
    Optional<GeneratedStory> story,
    Optional<QualityResult> quality,
    StoryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum StoryStatus {
    REQUESTED, ENRICHED, BLOCKED, GENERATING, STORY_RECORDED, SCORED, FAILED
}
```

Events on `StoryEntity`: `StoryRequested`, `PromptEnriched`, `StoryBlocked`, `GenerationStarted`, `StoryRecorded`, `QualityScored`, `StoryFailed`.

Every nullable lifecycle field on the `Story` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/stories` — body `{ genre, rawPrompt, constraints: { tone, wordCountTarget, pointOfView }, requestedBy }` → `{ storyId }`.
- `GET /api/stories` — list all stories, newest-first.
- `GET /api/stories/{id}` — one story.
- `GET /api/stories/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: StoryTeller</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted stories (status pill + quality score chip + age) and a right pane with the selected story's detail — original prompt, enriched prompt preview with content tags, story title, narrative body, author's note, and quality score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `StoryTellerAgent`. Asserts the candidate response is well-formed `GeneratedStory` JSON, `title` is non-empty, `body` is at least 150 characters, `genre` matches the submitted genre from the allowed set, and `authorNote` is non-empty. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `StoryRecorded` lands, as `qualityStep` inside the workflow. A deterministic scorer (no LLM call — the eval is rule-based on purpose) checks genre consistency, title-body alignment (key narrative words appear in the title or are referenced in the author note), and body structure (at least one paragraph break for stories over 200 words). Emits `QualityScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `StoryTellerAgent` → `prompts/story-teller.md`. The single decision-making LLM. System prompt instructs it to read the enriched prompt, honour the style constraints, and return a `GeneratedStory` with an informative author's note.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the fantasy seed prompt; within 30 s a well-formed story appears with a non-empty title, body, author's note, and a quality score chip.
2. **J2** — The agent's first response on a generation is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed story; the UI never displays the malformed response.
3. **J3** — A story generated with an empty `body` receives a quality score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A prompt containing a disallowed content category is submitted; the entity transitions to `BLOCKED`; no agent call is made; the UI shows the block reason.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named story-teller demonstrating the single-agent × content-editorial cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-content-editorial-story-teller. Java package io.akka.samples.storyteller.
Akka 3.6.0. HTTP port 9549.

Components to wire (exactly):

- 1 AutonomousAgent StoryTellerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/story-teller.md>) and
  .capability(TaskAcceptance.of(GENERATE_STORY).maxIterationsPerTask(3)). The task receives
  style constraints as its instruction text and the enriched prompt as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: GeneratedStory{title: String, body: String, authorNote: String, genre: String,
  generatedAt: Instant}. The agent is configured with a before-agent-response guardrail (see G1
  in eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries the response within its 3-iteration budget.

- 1 Workflow StoryWorkflow per storyId with three steps:
  * awaitEnrichedStep — polls StoryEntity.getStory every 1s; on story.enriched().isPresent()
    AND story.status == ENRICHED advances to generationStep; on story.status == BLOCKED
    transitions to the terminal done path with no agent call.
    WorkflowSettings.stepTimeout 15s.
  * generationStep — emits GenerationStarted, then calls componentClient.forAutonomousAgent(
    StoryTellerAgent.class, "teller-" + storyId).runSingleTask(
      TaskDef.instructions(formatConstraints(story.request.constraints))
        .attachment("prompt.txt", story.enriched.safePrompt.getBytes())
    ) — returns a taskId, then forTask(taskId).result(GENERATE_STORY) to fetch the story.
    On success calls StoryEntity.recordStory(generatedStory). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(StoryWorkflow::error).
  * qualityStep — runs a deterministic rule-based QualityScorer (NOT an LLM call) over the
    recorded story: checks that body is non-empty, that body has at least one paragraph break
    for word counts > 200, that genre appears consistent with the request, and that the
    author's note is informative (non-empty and at least 30 characters). Emits
    QualityScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity StoryEntity (one per storyId). State Story{storyId: String,
  request: Optional<StoryRequest>, enriched: Optional<EnrichedPrompt>,
  story: Optional<GeneratedStory>, quality: Optional<QualityResult>, status: StoryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. StoryStatus enum: REQUESTED, ENRICHED,
  BLOCKED, GENERATING, STORY_RECORDED, SCORED, FAILED. Events: StoryRequested{request},
  PromptEnriched{enriched}, StoryBlocked{reason}, GenerationStarted{}, StoryRecorded{story},
  QualityScored{quality}, StoryFailed{reason}. Commands: submit, attachEnriched, block,
  markGenerating, recordStory, recordQuality, fail, getStory. emptyState() returns
  Story.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer PromptEnricher subscribed to StoryEntity events; on StoryRequested runs a
  content-safety check (keyword-based blocklist for disallowed content categories: explicit
  violence, hate speech, explicit sexual content) over rawPrompt, computes content tags
  (genre markers, tone indicators, subject-matter tags), builds EnrichedPrompt; on pass calls
  StoryEntity.attachEnriched(enriched) and starts StoryWorkflow with id = "story-" + storyId;
  on fail calls StoryEntity.block(reason) and does NOT start the workflow.

- 1 View StoryView with row type StoryRow (mirrors Story minus request.rawPrompt — the audit
  log keeps the raw; the view holds the safe prompt for the UI). Table updater consumes
  StoryEntity events. ONE query getAllStories: SELECT * AS stories FROM story_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * StoryEndpoint at /api with POST /stories (body
    {genre, rawPrompt, constraints: {tone, wordCountTarget, pointOfView}, requestedBy};
    mints storyId; calls StoryEntity.submit; returns {storyId}), GET /stories
    (list from getAllStories, sorted newest-first), GET /stories/{id} (one row), GET
    /stories/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- StoryTasks.java declaring one Task<R> constant: GENERATE_STORY = Task.name("Generate
  story").description("Read the attached enriched prompt and produce a GeneratedStory matching
  the style constraints").resultConformsTo(GeneratedStory.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records StoryRequest, StyleConstraints, Tone, PointOfView, EnrichedPrompt,
  GeneratedStory, QualityResult, Story, StoryStatus.

- StoryGuardrail.java implementing the before-agent-response hook. Reads the candidate
  GeneratedStory from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: GeneratedStory and the
  submitted StoryRequest. Outputs: QualityResult. Scoring rubric documented in Javadoc on the
  class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9549 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The StoryTellerAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 3 seeded prompts: a fantasy quest
  seed (a hero discovers a map leading to a cursed library), a mystery investigation seed (a
  detective finds a locked room with no doors), and a sci-fi first-contact seed (a radio
  operator receives a signal from a planet that should be uninhabited).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with decisions.authority_level = full-automation
  (the agent's output IS the deliverable, not a recommendation to a human reviewer),
  oversight.human_in_loop = false, failure.failure_modes including
  "incoherent-narrative", "genre-mismatch", "truncated-body", "unsafe-content-generation";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/story-teller.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Story Teller", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of story cards with status pill + quality chip; right = selected-story detail with
  original prompt, enriched prompt preview + content tags, story title, narrative body,
  author's note, and quality score chip).
  Browser title exactly: <title>Akka Sample: StoryTeller</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
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
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-story.json — 8 GeneratedStory entries covering the three genre seeds plus
    custom. Each entry has a non-empty title, a body of at least 150 characters with at
    least one paragraph break, a non-empty authorNote, and a genre matching the requested
    genre. Plus 2 deliberately MALFORMED entries (one with an empty body; one with a genre
    value not in the allowed set) — the guardrail blocks both, exercising the retry path.
    The mock selects a malformed entry on the FIRST iteration of every 3rd story (modulo seed)
    so J2 is reproducible.
- A MockModelProvider.seedFor(storyId) helper makes per-story selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. StoryTellerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion StoryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (generationStep
  60s, awaitEnrichedStep 15s, qualityStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Story row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: StoryTasks.java with GENERATE_STORY = Task.name(...).description(...)
  .resultConformsTo(GeneratedStory.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9549 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (StoryTellerAgent). The
  on-decision quality eval is rule-based (QualityScorer.java) and does NOT make an LLM call.
- The prompt is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated generationStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
