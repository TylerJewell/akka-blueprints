# SPEC — blog-writer-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Blog Writer Pipeline.
**One-line pitch:** A user submits a topic and style preferences; one `BlogWriterAgent` walks it through three task phases — **RESEARCH** the topic space, **OUTLINE** a structured post, **DRAFT** the full article — with a brand-voice guardrail checking every draft response before the post is recorded, and a quality evaluator scoring every completed post.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain. One `BlogWriterAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RESEARCH task's typed output becomes the OUTLINE task's instruction context; the OUTLINE task's typed output becomes the DRAFT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and the recording of every draft output. It checks the candidate `BlogPost` against the brand-voice ruleset: minimum word count (≥ 300 words), absence of forbidden phrases (a configurable list loaded from `src/main/resources/brand-rules/forbidden-phrases.txt`), and presence of at least one call-to-action sentence in the final paragraph. A draft that fails any check is rejected before `DraftWritten` is recorded on `BlogPostEntity`; the rejection returns a structured error to the agent with the specific rule that failed, so the agent can revise within its iteration budget. The guardrail does NOT make an LLM call — the three checks are deterministic string-level rules. A separate on-decision evaluator (`QualityScorer`) scores the accepted draft on four content-quality dimensions; that score is informational and does not block recording.

The blueprint shows that the before-agent-response hook is the right cut to enforce brand-voice guarantees: checking the full generated response (not an individual tool call) against policy before it is committed to the event log.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **topic** into the input (or picks one of three seeded topics — `The future of developer tooling`, `How agentic AI changes product management`, `Practical guide to event sourcing`), and optionally selects a **style** from a dropdown (`Technical`, `Conversational`, `Thought leadership`).
2. The user clicks **Generate post**. The UI POSTs to `/api/posts` and receives a `postId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RESEARCHING` — the workflow has started `researchStep` and the agent has been handed the RESEARCH task.
4. Within ~10–20 s the card reaches `OUTLINING` — the typed `ResearchNotes` are visible in the card detail (a small table of reference points with source and key insight). The agent's RESEARCH task returned; the workflow recorded `ResearchCollected` and ran the OUTLINE task.
5. Within ~10–20 s more the card reaches `DRAFTING`. The `Outline` is visible (section list with headings and key points).
6. Within ~10–20 s more the card reaches `QUALITY_CHECKED`. The right pane now shows the full typed `BlogPost` — title, introduction, per-section blocks with body text, and a conclusion — plus a quality score chip (1–5) and a one-line rationale.
7. The user can submit another topic; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BlogPostEndpoint` | `HttpEndpoint` | `/api/posts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BlogPostEntity`, `BlogPostView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BlogPostEntity` | `EventSourcedEntity` | Per-post lifecycle: created → researching → researched → outlining → outlined → drafting → drafted → quality_checked. Source of truth. | `BlogPostEndpoint`, `BlogWritingWorkflow` | `BlogPostView` |
| `BlogWritingWorkflow` | `Workflow` | One workflow per post. Steps: `researchStep` → `outlineStep` → `draftStep` → `qualityCheckStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `BlogPostEndpoint` after `CREATED` | `BlogWriterAgent`, `BlogPostEntity` |
| `BlogWriterAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `BlogTasks.java`: `RESEARCH_TOPIC` → `ResearchNotes`, `OUTLINE_POST` → `Outline`, `DRAFT_POST` → `BlogPost`. Each task is registered with the phase-appropriate function tools. | invoked by `BlogWritingWorkflow` | returns typed results |
| `ResearchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchTopicReferences(topic)` and `fetchKeyPoints(url)`. Reads from `src/main/resources/sample-data/references/*.json` for deterministic offline output. | called from RESEARCH task | returns `List<Reference>` |
| `OutlineTools` | function-tools class | Implements `generateSectionHeadings(notes, style)` and `assignKeyPoints(headings, notes)`. Pure in-memory transformations. | called from OUTLINE task | returns `List<OutlineSection>` |
| `DraftTools` | function-tools class | Implements `writeSection(section, notes, style)` and `writeConclusion(sections, style)`. | called from DRAFT task | returns `PostSection` / `String` |
| `BrandGuardrail` | `before-agent-response` guardrail (registered on `BlogWriterAgent`) | Reads the candidate `BlogPost` response and applies the brand-voice ruleset: word count ≥ 300, no forbidden phrases, at least one CTA sentence in the final paragraph. On rejection, returns a structured `brand-violation` error naming the failing rule. | every agent response on the DRAFT task | accept / structured-reject |
| `QualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `BlogPost`, `Outline`, `ResearchNotes`. Output: `QualityResult{score, rationale}`. | called from `qualityCheckStep` | returns score |
| `BlogPostView` | `View` | Read model: one row per post for the UI. | `BlogPostEntity` events | `BlogPostEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Reference(String source, String url, String keyInsight, Instant capturedAt) {}

record ResearchNotes(List<Reference> references, String topicSummary, Instant researchedAt) {}

record OutlineSection(String sectionId, String heading, List<String> keyPoints) {}

record Outline(
    String proposedTitle,
    String introduction,
    List<OutlineSection> sections,
    Instant outlinedAt
) {}

record PostSection(String sectionId, String heading, String body) {}

record BlogPost(
    String title,
    String introduction,
    List<PostSection> sections,
    String conclusion,
    String style,
    Instant draftedAt
) {}

record QualityResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record GuardrailRejection(
    String rule,
    String detail,
    Instant rejectedAt
) {}

record BlogPostRecord(
    String postId,
    Optional<String> topic,
    Optional<String> style,
    Optional<ResearchNotes> research,
    Optional<Outline> outline,
    Optional<BlogPost> post,
    Optional<QualityResult> quality,
    List<GuardrailRejection> guardrailRejections,
    BlogPostStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BlogPostStatus {
    CREATED, RESEARCHING, RESEARCHED, OUTLINING, OUTLINED,
    DRAFTING, DRAFTED, QUALITY_CHECKED, FAILED
}
```

Events on `BlogPostEntity`: `PostCreated`, `ResearchStarted`, `ResearchCollected`, `OutlineStarted`, `OutlineProduced`, `DraftStarted`, `DraftWritten`, `QualityChecked`, `GuardrailRejected`, `PostFailed`.

Every nullable lifecycle field on the `BlogPostRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/posts` — body `{ topic, style }` → `{ postId }`.
- `GET /api/posts` — list all posts, newest-first.
- `GET /api/posts/{id}` — one post.
- `GET /api/posts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Blog Writer Pipeline</title>`.

The App UI tab is a two-column layout: a left rail with the live list of posts (status pill + topic + age) and a right pane with the selected post's detail — topic, style, research references table, outline sections, blog post sections, quality score chip, and a guardrail-rejection log strip if any brand-rule rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (brand-voice gate)**: `BrandGuardrail` is registered on `BlogWriterAgent` and runs before every agent response on the DRAFT task is committed to the event log. It reads the candidate `BlogPost` payload and applies three deterministic checks: (1) word count — the combined `introduction` + all `sections[].body` + `conclusion` text must be ≥ 300 words; (2) forbidden-phrase scan — the full post text must not contain any phrase from `src/main/resources/brand-rules/forbidden-phrases.txt`; (3) CTA presence — the `conclusion` must contain at least one sentence ending with a call-to-action marker ("learn more", "get started", "try", "contact us", "sign up", or a question mark). On rejection the guardrail returns a structured `brand-violation` error naming the failing rule and the specific failing text excerpt, so the agent can correct course within its 4-iteration budget. The workflow also records a `GuardrailRejected{rule, detail}` event for visibility in the UI's rejection-log strip and in the audit log.

- **E1 — `on-decision-eval`**: runs immediately after `DraftWritten` lands, as `qualityCheckStep` inside the workflow. `QualityScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every outline section must have a matching post section (section coverage), every post section must have a body of at least 50 words (depth), the post title must differ from the outline's `proposedTitle` by fewer than 20 characters (title fidelity), and the `sections.size()` must equal `outline.sections.size()` (section parity). Emits `QualityChecked{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `BlogWriterAgent` → `prompts/blog-writer-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded topic `The future of developer tooling` with style `Technical`; within 60 s the post reaches `QUALITY_CHECKED` with non-empty references, ≥ 2 outline sections, ≥ 2 post sections, and a quality score chip on the card.
2. **J2** — The agent's first DRAFT iteration produces a post under the 300-word minimum (mock LLM path). `BrandGuardrail` rejects it; a `GuardrailRejected` event lands on the entity; the agent retries with a longer draft; the post eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A post whose mock-LLM trajectory produces a draft containing a forbidden phrase is rejected by the guardrail; the agent removes the phrase in retry; the final post passes the check and is recorded. The UI shows the rejection before the accepted draft appears.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-post trace (logged at `INFO`); the RESEARCH task's log shows only RESEARCH-tool calls, the OUTLINE task's log shows only OUTLINE-tool calls, the DRAFT task's log shows only DRAFT-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named blog-writer-pipeline demonstrating the sequential-pipeline x
content-editorial cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact sequential-pipeline-content-editorial-blog-writer-pipeline.
Java package io.akka.samples.blogwriter. Akka 3.6.0. HTTP port 9250.

Components to wire (exactly):

- 1 AutonomousAgent BlogWriterAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/blog-writer-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the RESEARCH, OUTLINE, and DRAFT tool sets are ALL registered on the
  agent; task scoping is handled by the agent's task context, not by conditional .tools(...)
  wiring. The before-agent-response guardrail (BrandGuardrail) is registered on the agent
  via the agent's guardrail-configuration block. On guardrail rejection the agent loop retries
  within its 4-iteration budget.

- 1 Workflow BlogWritingWorkflow per postId with four steps:
  * researchStep — emits ResearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(BlogWriterAgent.class, "agent-" + postId).runSingleTask(
      TaskDef.instructions("Topic: " + topic + "\nStyle: " + style + "\nPhase: RESEARCH\n
      Use searchTopicReferences and fetchKeyPoints to gather 4-8 references about this topic.")
        .metadata("postId", postId)
        .metadata("phase", "RESEARCH")
        .taskType(BlogTasks.RESEARCH_TOPIC)
    ). Reads forTask(taskId).result(RESEARCH_TOPIC) to get ResearchNotes. Writes
    BlogPostEntity.recordResearch(notes). WorkflowSettings.stepTimeout 60s.
  * outlineStep — emits OutlineStarted, then runSingleTask with TaskDef.instructions
    (formatOutlineContext(notes, topic, style)) and metadata.phase = "OUTLINE", taskType
    OUTLINE_POST. Writes BlogPostEntity.recordOutline(outline). stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(outline, notes, topic, style)) and metadata.phase = "DRAFT", taskType
    DRAFT_POST. Writes BlogPostEntity.recordDraft(post). stepTimeout 90s (longer to
    accommodate guardrail retry on brand violations).
  * qualityCheckStep — runs the deterministic QualityScorer over (post, outline, notes)
    and writes BlogPostEntity.recordQuality(quality). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(BlogWritingWorkflow::error). The error step writes PostFailed
  and ends.

- 1 EventSourcedEntity BlogPostEntity (one per postId). State BlogPostRecord{postId,
  topic: Optional<String>, style: Optional<String>, research: Optional<ResearchNotes>,
  outline: Optional<Outline>, post: Optional<BlogPost>, quality: Optional<QualityResult>,
  guardrailRejections: List<GuardrailRejection>, status: BlogPostStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. BlogPostStatus enum: CREATED, RESEARCHING, RESEARCHED,
  OUTLINING, OUTLINED, DRAFTING, DRAFTED, QUALITY_CHECKED, FAILED. Events:
  PostCreated{topic, style}, ResearchStarted, ResearchCollected{research}, OutlineStarted,
  OutlineProduced{outline}, DraftStarted, DraftWritten{post}, QualityChecked{quality},
  GuardrailRejected{rule, detail, rejectedAt}, PostFailed{reason}.
  Commands: create, startResearch, recordResearch, startOutline, recordOutline, startDraft,
  recordDraft, recordQuality, recordGuardrailRejection, fail, getPost. emptyState() returns
  BlogPostRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View BlogPostView with row type BlogPostRow that mirrors BlogPostRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes BlogPostEntity events. ONE
  query getAllPosts: SELECT * AS posts FROM blog_post_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * BlogPostEndpoint at /api with POST /posts (body {topic, style}; mints postId; calls
    BlogPostEntity.create(topic, style); then starts BlogWritingWorkflow with id
    "workflow-" + postId; returns {postId}), GET /posts (list from getAllPosts,
    sorted newest-first), GET /posts/{id} (one row), GET /posts/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- BlogTasks.java declaring three Task<R> constants:
    RESEARCH_TOPIC = Task.name("Research topic").description("Gather references and key
      insights about a topic by calling searchTopicReferences and fetchKeyPoints")
      .resultConformsTo(ResearchNotes.class);
    OUTLINE_POST = Task.name("Outline post").description("Generate section headings and
      assign key points from research notes into a structured Outline")
      .resultConformsTo(Outline.class);
    DRAFT_POST = Task.name("Draft post").description("Write a full BlogPost whose
      PostSections mirror the Outline sections one-to-one")
      .resultConformsTo(BlogPost.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- ResearchTools.java — @FunctionTool searchTopicReferences(String topic) -> List<Reference>
  reading from src/main/resources/sample-data/references/*.json keyed by topic; @FunctionTool
  fetchKeyPoints(String url) -> String reading from the matching reference entry's keyInsight.

- OutlineTools.java — @FunctionTool generateSectionHeadings(ResearchNotes notes, String style)
  -> List<OutlineSection> (deterministic mapping — one section per 2 references, capped at
  5 sections, sectionId minted as "s-" + index); @FunctionTool assignKeyPoints(
  List<OutlineSection> headings, ResearchNotes notes) -> List<OutlineSection> (enriches each
  section with 2-3 keyPoints drawn from matching reference keyInsights).

- DraftTools.java — @FunctionTool writeSection(OutlineSection section, ResearchNotes notes,
  String style) -> PostSection (body composed from the section's keyPoints joined with
  style-appropriate linking phrases); @FunctionTool writeConclusion(List<PostSection> sections,
  String style) -> String (1-paragraph conclusion ending with a CTA sentence).

- BrandGuardrail.java — implements the before-agent-response hook. Reads the candidate
  BlogPost response payload. Applies three checks:
  (1) word count: combined introduction + all sections[].body + conclusion >= 300 words.
  (2) forbidden-phrase scan: post text does not contain any phrase from
  src/main/resources/brand-rules/forbidden-phrases.txt (one phrase per line, case-insensitive).
  (3) CTA presence: conclusion contains at least one sentence with a CTA marker.
  On any failure: returns Guardrail.reject("brand-violation: <rule> — <detail>") AND calls
  BlogPostEntity.recordGuardrailRejection(rule, detail, now) so the rejection is visible
  in the UI's rejection-log strip and in the audit log. On accept: passes unchanged.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: BlogPost, Outline,
  ResearchNotes. Outputs: QualityResult with score and rationale. Four checks, one point
  per check satisfied, starting from a base of 1: section coverage (every OutlineSection
  has a matching PostSection by sectionId), section depth (every PostSection.body >= 50
  words), title fidelity (Levenshtein distance between BlogPost.title and
  Outline.proposedTitle <= 20), and section parity (post.sections.size() ==
  outline.sections.size()). Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9250 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/topics.jsonl with 5 seeded topic lines.

- src/main/resources/sample-data/references/*.json — three files keyed by seeded topic,
  each carrying 6-10 Reference entries with deterministic content so
  ResearchTools.searchTopicReferences returns the same list across restarts.

- src/main/resources/brand-rules/forbidden-phrases.txt — 10-15 phrases (e.g.
  "synergize", "paradigm shift", "use our", "game-changer", "disruptive innovation",
  "thought leadership" when used as self-description). One phrase per line.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root with data.data_classes.pii = false (topics and
  references are content-level, not person-level), decisions.authority_level = recommend-only
  (the blog post is editorial output requiring human review before publishing),
  oversight.human_in_loop = true (an editor reviews the post before it is published),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "brand-voice-violation", "section-depth-insufficient",
  "forbidden-phrase", "missing-cta", "title-drift"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/blog-writer-agent.md loaded as the agent system prompt.

- README.md at the project root (matching this file's content).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of post cards; right = selected-post detail with topic header, research references
  table, outline sections, blog post sections with body text, quality-score chip, rejection-log
  strip). Browser title exactly: <title>Akka Sample: Blog Writer Pipeline</title>.
  No subtitle on the Overview tab.

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

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/<task-id>.json,
  picks one entry pseudo-randomly per call (seedFor(postId)), and deserialises into the task's
  typed return. Each entry carries a "tool_calls" array the mock replays in order.
- Per-task mock-response shapes:
    research-topic.json — 6 ResearchNotes entries, each with 5-8 Reference items per seeded
      topic. Each entry's tool_calls: 1 searchTopicReferences(topic) + 1-3 fetchKeyPoints(url).
    outline-post.json — 6 Outline entries paired one-to-one with research entries, each with
      2-4 OutlineSection items, tool_calls: generateSectionHeadings + assignKeyPoints.
    draft-post.json — 6 BlogPost entries paired one-to-one. Each with 2-4 PostSection items
      matching the paired Outline sections, tool_calls: writeSection (one per section) +
      writeConclusion. Plus 1 deliberately SHORT entry whose combined word count is < 300 —
      BrandGuardrail rejects it; the mock then falls through to a normal draft. The mock
      selects the short entry on the FIRST iteration of every 3rd post (modulo seed) so J2
      is reproducible. Plus 1 deliberately FORBIDDEN-PHRASE entry containing "paradigm shift"
      in the introduction — BrandGuardrail rejects it; the mock provides a clean retry.
- A MockModelProvider.seedFor(postId) helper makes per-post selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BlogWriterAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BlogTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (researchStep
  60s, outlineStep 60s, draftStep 90s, qualityCheckStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on BlogPostRecord is Optional<T>.
- Lesson 7: BlogTasks.java with RESEARCH_TOPIC, OUTLINE_POST, DRAFT_POST constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9250 declared explicitly in application.conf's akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BlogWriterAgent). The
  on-decision eval is rule-based (QualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent; the
  task context carries the current phase; the agent must call tools matching the current phase.
- Task dependency is carried by typed task results: researchStep writes ResearchNotes onto
  the entity, outlineStep reads it and builds the OUTLINE task's instruction context from it,
  draftStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build". Per Lesson 25, /akka:specify
  handles the key during generation.
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
