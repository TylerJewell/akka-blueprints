# SPEC — AI Blog Writer Pipeline with Ollama

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AI Blog Writer Pipeline with Ollama.
**One-line pitch:** A user submits a topic; one `BlogWriterAgent` walks it through five task phases — **RESEARCH** reference material, **OUTLINE** the structure, **DRAFT** the prose, **EDIT** for clarity and tone, **PUBLISH** the final post — with each phase gated on the prior phase's recorded output and a content-policy guardrail checking prose before it advances.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain. One `BlogWriterAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RESEARCH task's typed output becomes the OUTLINE task's instruction context; the OUTLINE task's typed output becomes the DRAFT task's instruction context; the DRAFT task's typed output becomes the EDIT task's instruction context; the EDIT task's typed output becomes the PUBLISH task's instruction context. The agent never holds all five phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and the workflow step that records each phase's result. Before the workflow writes the result onto `PostEntity`, the guardrail scans the prose for policy violations (prohibited topics, keyword patterns representing plagiarism risk, and content-category mismatches for the declared post type). A violation returns a structured rejection to the agent loop so the task can retry inside its iteration budget. The same hook is the runtime proof that no unreviewed prose reaches the stored entity or the published UI card.

The blueprint shows that a sequential pipeline for editorial content is not just a chain of LLM calls — the task-boundary handoffs carry the editorial dependency (outline constrains draft; draft constrains edit; edit constrains publish) while the guardrail carries the cross-cutting policy contract.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **topic** into the input (or picks one of three seeded topics — `Getting started with reactive systems`, `How event sourcing simplifies audit trails`, `Building a local AI pipeline without cloud dependencies`).
2. The user picks a **post type** (`tutorial`, `explainer`, `opinion`) and clicks **Run pipeline**. The UI POSTs to `/api/posts` and receives a `postId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RESEARCHING` — the workflow has started `researchStep` and the agent has been handed the RESEARCH task.
4. Within ~15–25 s the card reaches `OUTLINED` — the typed `ResearchNotes` are visible in the card detail (a small table of reference points with source and summary). The agent's RESEARCH task returned; the workflow recorded `ResearchCollected` and ran the OUTLINE task.
5. Within ~15–25 s more the card reaches `DRAFTED`. The `Outline` is visible (section list + key points per section).
6. Within ~15–25 s more the card reaches `EDITED`. The `Draft` is visible (title, sections with prose paragraphs).
7. Within ~15–25 s more the card reaches `PUBLISHED`. The right pane now shows the full typed `Post` — title, summary, per-section `PostSection` with body paragraphs and a `references` list — plus a policy-clear status chip.
8. The user can submit another topic; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PostEndpoint` | `HttpEndpoint` | `/api/posts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PostEntity`, `PostView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PostEntity` | `EventSourcedEntity` | Per-post lifecycle: created → researching → researched → outlining → outlined → drafting → drafted → editing → edited → publishing → published / failed. Source of truth. | `PostEndpoint`, `BlogPipelineWorkflow` | `PostView` |
| `BlogPipelineWorkflow` | `Workflow` | One workflow per post. Steps: `researchStep` → `outlineStep` → `draftStep` → `editStep` → `publishStep`. Each step runs one task on the agent, reads the typed result, runs the content-policy guardrail check, writes the corresponding event onto the entity, then advances. | started by `PostEndpoint` after `CREATED` | `BlogWriterAgent`, `PostEntity` |
| `BlogWriterAgent` | `AutonomousAgent` | The single agent. Declares five `Task<R>` constants in `BlogTasks.java`: `RESEARCH_TOPIC` → `ResearchNotes`, `OUTLINE_POST` → `Outline`, `DRAFT_POST` → `Draft`, `EDIT_POST` → `EditedDraft`, `PUBLISH_POST` → `Post`. Each task is registered with the phase-appropriate function tools. | invoked by `BlogPipelineWorkflow` | returns typed results |
| `ResearchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchReferences(topic)` and `fetchSummary(url)`. Reads from `src/main/resources/sample-data/references/*.json` for deterministic offline output. | called from RESEARCH task | returns `List<Reference>` |
| `OutlineTools` | function-tools class | Implements `structureSections(notes)` and `expandKeyPoints(section)`. Pure in-memory transformations. | called from OUTLINE task | returns `List<OutlineSection>` |
| `DraftTools` | function-tools class | Implements `writeParagraph(section, keyPoints)` and `composeTitle(outline)`. | called from DRAFT task | returns `DraftSection` / `String` |
| `EditTools` | function-tools class | Implements `applyToneAdjustments(draft, postType)` and `checkReadability(section)`. | called from EDIT task | returns `EditedSection` / `ReadabilityScore` |
| `PublishTools` | function-tools class | Implements `formatPost(editedDraft, postType)` and `collectReferences(notes)`. | called from PUBLISH task | returns `Post` / `List<PostReference>` |
| `ContentPolicyGuardrail` | `before-agent-response` guardrail (registered on `BlogWriterAgent`) | Scans the candidate prose from each phase for prohibited content patterns, checks originality heuristics, and verifies post-type alignment. Rejects non-compliant prose before it is written onto the entity. | every agent response on every task | accept / structured-reject |
| `PostView` | `View` | Read model: one row per post for the UI. | `PostEntity` events | `PostEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Reference(String source, String url, String summary, Instant fetchedAt) {}

record ResearchNotes(List<Reference> references, String topicSummary, Instant researchedAt) {}

record OutlineSection(String sectionId, String heading, List<String> keyPoints) {}

record Outline(String title, List<OutlineSection> sections, Instant outlinedAt) {}

record DraftSection(String sectionId, String heading, String body) {}

record Draft(String title, String introduction, List<DraftSection> sections, Instant draftedAt) {}

record EditedSection(String sectionId, String heading, String body, ReadabilityScore readability) {}

record ReadabilityScore(int fleschKincaid, String toneLabel) {}

record EditedDraft(
    String title,
    String introduction,
    List<EditedSection> sections,
    Instant editedAt
) {}

record PostReference(String label, String url) {}

record PostSection(String sectionId, String heading, String body, List<PostReference> references) {}

record Post(
    String title,
    String summary,
    String postType,
    List<PostSection> sections,
    Instant publishedAt
) {}

record PolicyCheckResult(
    boolean passed,
    String reason,
    Instant checkedAt
) {}

record PostRecord(
    String postId,
    Optional<String> topic,
    Optional<String> postType,
    Optional<ResearchNotes> research,
    Optional<Outline> outline,
    Optional<Draft> draft,
    Optional<EditedDraft> editedDraft,
    Optional<Post> post,
    Optional<PolicyCheckResult> lastPolicyCheck,
    PostStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PostStatus {
    CREATED, RESEARCHING, RESEARCHED, OUTLINING, OUTLINED, DRAFTING,
    DRAFTED, EDITING, EDITED, PUBLISHING, PUBLISHED, FAILED
}
```

Events on `PostEntity`: `PostCreated`, `ResearchStarted`, `ResearchCollected`, `OutlineStarted`, `OutlineProduced`, `DraftStarted`, `DraftWritten`, `EditStarted`, `EditApplied`, `PublishStarted`, `ContentCleared`, `PostPublished`, `GuardrailBlocked`, `PostFailed`.

Every nullable lifecycle field on the `PostRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/posts` — body `{ topic, postType }` → `{ postId }`.
- `GET /api/posts` — list all posts, newest-first.
- `GET /api/posts/{id}` — one post.
- `GET /api/posts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AI Blog Writer Pipeline with Ollama</title>`.

The App UI tab is a two-column layout: a left rail with the live list of posts (status pill + topic + age) and a right pane with the selected post's detail — topic, research notes table, outline sections, draft text, edited draft, published post sections, policy check status, and a guardrail-block log strip if any phase-gate blocks occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (content-policy)**: `ContentPolicyGuardrail` is registered on `BlogWriterAgent` and runs before every agent task response is written onto the entity. It receives the full prose output of the completed task, scans for prohibited content patterns (a configurable keyword list loaded from `src/main/resources/policy/prohibited-patterns.json`), applies an originality heuristic (checks that every factual claim in the prose is anchored to a `Reference.url` from the upstream `ResearchNotes`), and verifies post-type alignment (an `opinion` post must contain first-person stance markers; a `tutorial` must contain at least one code fence). On violation, the guardrail returns a structured `policy-violation` error to the agent loop and the workflow records a `GuardrailBlocked{phase, reason, checkedAt}` event for visibility. The agent loop retries within its 4-iteration budget. On accept, the workflow writes `ContentCleared{phase, checkedAt}` and advances.

## 9. Agent prompts

- `BlogWriterAgent` → `prompts/blog-writer-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. It also instructs the agent that a `policy-violation` rejection means the prose must be revised — not that the topic should be abandoned.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded topic `Getting started with reactive systems` with type `tutorial`; within 90 s the post reaches `PUBLISHED` with non-empty references, ≥ 3 sections, and a policy-clear status chip on the card.
2. **J2** — The mock LLM's first DRAFT iteration produces prose containing a prohibited keyword. `ContentPolicyGuardrail` blocks the response; a `GuardrailBlocked` event lands on the entity; the agent retries with revised prose; the post eventually reaches `PUBLISHED`. The UI's block-log strip shows the one blocked response.
3. **J3** — A post whose mock-LLM trajectory produces a `tutorial` typed post with no code fence is blocked by the guardrail's post-type alignment check; the agent retries with a corrected draft; the final post passes.
4. **J4** — Each task's instructions, tool calls, and prose output are visible in the per-post trace (logged at `INFO`); the RESEARCH task's log shows only RESEARCH-tool calls, the OUTLINE task's log shows only OUTLINE-tool calls, and so on through PUBLISH. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ai-blog-writer-pipeline-with-ollama demonstrating the
sequential-pipeline x content-editorial cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-content-editorial-blog-pipeline. Java package
io.akka.samples.aiblogwriterpipelinewithollama. Akka 3.6.0. HTTP port 9628.

Components to wire (exactly):

- 1 AutonomousAgent BlogWriterAgent — extends
  akka.javasdk.agent.autonomous.AutonomousAgent. definition() returns
  AgentDefinition with .instructions(<system prompt loaded from
  prompts/blog-writer-agent.md>) and five .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are
  registered with .tools(...) — ALL five tool sets are registered on the agent;
  phase ordering is the job of the workflow, NOT of conditional .tools(...) wiring.
  The before-agent-response guardrail (ContentPolicyGuardrail) is registered on
  the agent via the agent's guardrail-configuration block. On guardrail rejection
  the agent loop retries within its 4-iteration budget.

- 1 Workflow BlogPipelineWorkflow per postId with five steps plus an error step:
  * researchStep — emits ResearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(BlogWriterAgent.class, "agent-" + postId).runSingleTask(
      TaskDef.instructions("Topic: " + topic + "\nPostType: " + postType +
      "\nPhase: RESEARCH\nUse the searchReferences and fetchSummary tools to
      collect 4-8 reference items about this topic.")
        .metadata("postId", postId)
        .metadata("phase", "RESEARCH")
        .taskType(BlogTasks.RESEARCH_TOPIC)
    ). Reads forTask(taskId).result(RESEARCH_TOPIC) to get ResearchNotes. Writes
    PostEntity.recordResearch(research). WorkflowSettings.stepTimeout 90s.
  * outlineStep — emits OutlineStarted, then runSingleTask with TaskDef.instructions
    (formatOutlineContext(research, topic, postType)) and metadata.phase = "OUTLINE",
    taskType OUTLINE_POST. Writes PostEntity.recordOutline(outline). stepTimeout 90s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(outline, research, topic, postType)) and metadata.phase =
    "DRAFT", taskType DRAFT_POST. Writes PostEntity.recordDraft(draft).
    stepTimeout 90s.
  * editStep — emits EditStarted, then runSingleTask with TaskDef.instructions
    (formatEditContext(draft, postType)) and metadata.phase = "EDIT", taskType
    EDIT_POST. Writes PostEntity.recordEditedDraft(editedDraft). stepTimeout 90s.
  * publishStep — emits PublishStarted, then runSingleTask with TaskDef.instructions
    (formatPublishContext(editedDraft, research, topic, postType)) and
    metadata.phase = "PUBLISH", taskType PUBLISH_POST. Writes
    PostEntity.recordPost(post). stepTimeout 90s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a
  top-level WorkflowSettings class (Lesson 5). settings() also declares
  defaultStepRecovery maxRetries(2).failoverTo(BlogPipelineWorkflow::error). The
  error step writes PostFailed and ends.

- 1 EventSourcedEntity PostEntity (one per postId). State PostRecord{postId,
  topic: Optional<String>, postType: Optional<String>,
  research: Optional<ResearchNotes>, outline: Optional<Outline>,
  draft: Optional<Draft>, editedDraft: Optional<EditedDraft>,
  post: Optional<Post>, lastPolicyCheck: Optional<PolicyCheckResult>,
  status: PostStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  PostStatus enum: CREATED, RESEARCHING, RESEARCHED, OUTLINING, OUTLINED,
  DRAFTING, DRAFTED, EDITING, EDITED, PUBLISHING, PUBLISHED, FAILED. Events:
  PostCreated{topic, postType}, ResearchStarted, ResearchCollected{research},
  OutlineStarted, OutlineProduced{outline}, DraftStarted, DraftWritten{draft},
  EditStarted, EditApplied{editedDraft}, PublishStarted,
  ContentCleared{phase, checkedAt}, PostPublished{post},
  GuardrailBlocked{phase, reason, checkedAt}, PostFailed{reason}.
  Commands: create, startResearch, recordResearch, startOutline, recordOutline,
  startDraft, recordDraft, startEdit, recordEditedDraft, startPublish,
  recordContentCleared, recordPost, recordGuardrailBlock, fail, getPost.
  emptyState() returns PostRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 View PostView with row type PostRow that mirrors PostRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes PostEntity
  events. ONE query getAllPosts: SELECT * AS posts FROM post_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * PostEndpoint at /api with POST /posts (body {topic, postType}; mints postId;
    calls PostEntity.create(topic, postType); then starts BlogPipelineWorkflow
    with id "pipeline-" + postId; returns {postId}), GET /posts (list from
    getAllPosts, sorted newest-first), GET /posts/{id} (one row), GET /posts/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion classes:

- BlogTasks.java declaring five Task<R> constants:
    RESEARCH_TOPIC = Task.name("Research topic").description("Gather reference
      material about a topic by calling searchReferences and fetchSummary")
      .resultConformsTo(ResearchNotes.class);
    OUTLINE_POST = Task.name("Outline post").description("Structure the post into
      sections with key points by calling structureSections and expandKeyPoints")
      .resultConformsTo(Outline.class);
    DRAFT_POST = Task.name("Draft post").description("Write prose for each section
      by calling writeParagraph and composeTile")
      .resultConformsTo(Draft.class);
    EDIT_POST = Task.name("Edit post").description("Apply tone adjustments and
      check readability by calling applyToneAdjustments and checkReadability")
      .resultConformsTo(EditedDraft.class);
    PUBLISH_POST = Task.name("Publish post").description("Format the final post
      and collect references by calling formatPost and collectReferences")
      .resultConformsTo(Post.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class
  (Lesson 7).

- ResearchTools.java — @FunctionTool searchReferences(String topic) ->
  List<Reference> reading from src/main/resources/sample-data/references/*.json
  keyed by topic; @FunctionTool fetchSummary(String url) -> String reading from
  the matching reference entry's summary.

- OutlineTools.java — @FunctionTool structureSections(ResearchNotes notes) ->
  List<OutlineSection> (3-5 sections, headings derived from key themes in the
  notes); @FunctionTool expandKeyPoints(OutlineSection section) ->
  List<String> (2-4 key points per section).

- DraftTools.java — @FunctionTool writeParagraph(OutlineSection section,
  List<String> keyPoints) -> DraftSection (prose paragraph from key points);
  @FunctionTool composeTile(Outline outline) -> String (post title).

- EditTools.java — @FunctionTool applyToneAdjustments(DraftSection section,
  String postType) -> EditedSection (adjusted body + tone label);
  @FunctionTool checkReadability(EditedSection section) -> ReadabilityScore
  (fleschKincaid score + tone label).

- PublishTools.java — @FunctionTool formatPost(EditedDraft draft, String
  postType) -> Post (assembled final post with title, summary, sections);
  @FunctionTool collectReferences(ResearchNotes notes) -> List<PostReference>
  (one PostReference per distinct reference entry).

- ContentPolicyGuardrail.java — implements the before-agent-response hook. Receives
  the completed prose from the agent's task response. Loads prohibited patterns from
  src/main/resources/policy/prohibited-patterns.json and scans prose for matches.
  Runs originality check: every factual claim in DRAFT/EDIT/PUBLISH phase prose
  must trace to a Reference.url in the upstream ResearchNotes (passed in via
  TaskDef metadata "researchNotesJson"). Runs post-type alignment check: a
  "tutorial" post must contain at least one code fence (```); an "opinion" post
  must contain first-person stance markers ("I believe", "In my view", "My take").
  On violation returns Guardrail.reject("policy-violation: <rule> failed — <detail>").
  On violation ALSO calls PostEntity.recordGuardrailBlock(phase, reason, checkedAt)
  so the block is visible in the UI's block-log strip and in the audit log. On
  accept calls PostEntity.recordContentCleared(phase, checkedAt).

- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9628 and the three model-provider blocks
  (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash)
  reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. If none is set at startup, fall back to ollama
  llama3.2 on localhost:11434.

- src/main/resources/sample-events/topics.jsonl with 5 seeded topic lines covering
  the three topic surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/references/*.json — three files keyed by seeded
  topic, each carrying 6-8 Reference entries with deterministic content so
  ResearchTools.searchReferences returns the same list across restarts.

- src/main/resources/policy/prohibited-patterns.json — array of keyword pattern
  objects with "pattern" and "reason" fields. At minimum includes patterns for
  demonstrating the J2 guardrail block path.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism
  in Section 8 of this SPEC. Matching simplified_view list.
  No regulation_anchors — general editorial domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (research
  references are topic-level, not person-level), decisions.authority_level =
  recommend-only (the post is editorial content), oversight.human_in_loop = true
  (a human editor reviews the post before final distribution), operations.agent_count
  = 1, operations.agent_pattern = sequential-pipeline, failure.failure_modes
  including "prohibited-content", "post-type-misalignment", "unsupported-claims",
  "originality-violation"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/blog-writer-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: AI Blog Writer Pipeline with
  Ollama", prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a
  two-column layout (left = live list of post cards; right = selected-post detail
  with topic header, research-notes table, outline sections, draft text, edited
  draft, published post, policy-check status chip, block-log strip). Browser title
  exactly: <title>Akka Sample: AI Blog Writer Pipeline with Ollama</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        deterministic random-but-typed-correct outputs per Task (see Mock LLM
        provider block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via
        the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time
        using the matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone when the
        session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the user's
  existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime. The message must not echo any
  captured key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(postId)), and deserialises into the task's typed return. Within
  a single task run, the mock also drives tool-call sequences — each entry carries
  a "tool_calls" array the mock replays in order before returning the final typed
  result.
- Per-task mock-response shapes for THIS blueprint:
    research-topic.json — 5 ResearchNotes entries, each with 5-7 Reference items.
      Each entry's tool_calls array contains 2-3 calls: 1 searchReferences(topic)
      + 1-2 fetchSummary(url) calls.
    outline-post.json — 5 Outline entries paired one-to-one with the research
      entries, each with 3-4 OutlineSection items and tool_calls containing
      structureSections + expandKeyPoints in order.
    draft-post.json — 5 Draft entries paired one-to-one with the outline entries.
      Each entry carries 3-4 DraftSection items, tool_calls containing writeParagraph
      (one per section) + composeTile. Plus 1 deliberately POLICY-VIOLATING entry
      whose prose contains a prohibited keyword — the guardrail blocks it, the mock
      then falls through to a clean draft. The mock should select the violating entry
      on the FIRST iteration of every 3rd post (modulo seed) so J2 is reproducible.
    edit-post.json — 5 EditedDraft entries paired one-to-one, each carrying 3-4
      EditedSection items with tone labels, tool_calls containing
      applyToneAdjustments + checkReadability in order.
    publish-post.json — 5 Post entries paired one-to-one, each carrying 3-4
      PostSection items with references, tool_calls containing formatPost +
      collectReferences. Plus 1 post-type-misalignment entry whose tutorial post
      contains no code fence — the guardrail's post-type alignment check blocks it;
      J3 verifies this.
- A MockModelProvider.seedFor(postId) helper makes per-post selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BlogWriterAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion
  BlogTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (researchStep 90s, outlineStep 90s, draftStep 90s, editStep 90s, publishStep
  90s, error 5s).
- Lesson 6: every nullable lifecycle field on the PostRecord row record is
  Optional<T>. The view table updater wraps values with Optional.of(...); callers
  use .orElse(...) or .isPresent().
- Lesson 7: BlogTasks.java with RESEARCH_TOPIC, OUTLINE_POST, DRAFT_POST,
  EDIT_POST, PUBLISH_POST constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9628 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black
  and arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records
  only the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel"> elements
  — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BlogWriterAgent).
  The content-policy guardrail is rule-based (ContentPolicyGuardrail.java) and does
  NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the
  agent; the workflow enforces phase order by advancing steps only after the prior
  step's typed result has been written onto the entity. The guardrail enforces the
  content policy across every phase response.
- Task dependency is carried by typed task results: researchStep writes
  ResearchNotes onto the entity, outlineStep reads it and builds the OUTLINE task's
  instruction context from it, draftStep reads both, editStep reads the draft, and
  publishStep reads the editedDraft and research. The agent itself is stateless
  across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
