# SPEC — video-to-blog-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** YouTube to Blog.
**One-line pitch:** A user submits a YouTube URL; one `BlogAgent` walks it through four task phases — **TRANSCRIPT** the video, **SUMMARISE** into structured themes, **DRAFT** a blog post, **POLISH** the prose — with each phase gated on the prior phase's recorded output, a `before-agent-response` guardrail screening the final output, and a deterministic quality eval scoring every post.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain. One `BlogAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the TRANSCRIPT task's typed output becomes the SUMMARISE task's instruction context; the SUMMARISE task's typed output becomes the DRAFT task's instruction context; the DRAFT task's typed output becomes the POLISH task's instruction context. The agent never holds all four phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and the final POLISH task's response. After the agent composes the polished blog post but before `PostPolished` is recorded, `PublishGuardrail` scans the response text for prohibited content patterns (hardcoded list: harmful-health-advice phrases, affiliate spam markers, deceptive-disclosure phrases) and applies brand-safety rules (no competitor brand mentions in body text). On reject, the guardrail returns a structured error to the agent loop and the workflow records a `GuardrailRejected{phase, reason}` event for visibility. The agent loop retries within its 3-iteration budget on the POLISH task.
- An **`on-decision-eval`** runs immediately after `PostPolished` lands, as `evalStep` inside the workflow. A deterministic, rule-based `EditorialScorer` (no LLM call — the eval is rule-based on purpose, so the same post always scores the same) checks that the draft section count equals the post section count (structural parity), that every post section has a heading, that the post's word count falls within the target range (400–1200 words), and that each section body is non-empty. Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce the dependency contract, and the guardrail is the right cut for final-output moderation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a **YouTube URL** into the input (or picks one of three seeded URLs — `https://www.youtube.com/watch?v=seed-akka-agents`, `https://www.youtube.com/watch?v=seed-llm-pricing`, `https://www.youtube.com/watch?v=seed-event-sourcing`).
2. The user clicks **Convert to blog post**. The UI POSTs to `/api/posts` and receives a `postId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `TRANSCRIBING` — the workflow has started `transcriptStep` and the agent has been handed the TRANSCRIPT task.
4. Within ~10–20 s the card reaches `SUMMARISING` — the typed `Transcript` is visible in the card detail (text preview, duration, chapter list). The agent's TRANSCRIPT task returned; the workflow recorded `TranscriptExtracted` and started the SUMMARISE task.
5. Within ~10–20 s more the card reaches `DRAFTING`. The `VideoSummary` is visible (key-point list + section outline with headings).
6. Within ~10–20 s more the card reaches `POLISHING`. The `BlogDraft` is visible (raw section bodies before prose polish).
7. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `BlogPost` — title, introduction, per-section blocks with heading, body, and a `sourceTimestamps` list — plus an eval score chip (1–5) and a one-line rationale.
8. The user can submit another URL; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BlogEndpoint` | `HttpEndpoint` | `/api/posts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BlogPostEntity`, `BlogPostView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BlogPostEntity` | `EventSourcedEntity` | Per-post lifecycle: created → transcribing → transcribed → summarising → summarised → drafting → drafted → polishing → polished → evaluated. Source of truth. | `BlogEndpoint`, `BlogPipelineWorkflow` | `BlogPostView` |
| `BlogPipelineWorkflow` | `Workflow` | One workflow per postId. Steps: `transcriptStep` → `summaryStep` → `draftStep` → `polishStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `BlogEndpoint` after `CREATED` | `BlogAgent`, `BlogPostEntity` |
| `BlogAgent` | `AutonomousAgent` | The single agent. Declares four `Task<R>` constants in `BlogTasks.java`: `EXTRACT_TRANSCRIPT` → `Transcript`, `SUMMARISE_VIDEO` → `VideoSummary`, `DRAFT_POST` → `BlogDraft`, `POLISH_POST` → `BlogPost`. Each task is registered with the phase-appropriate function tools. | invoked by `BlogPipelineWorkflow` | returns typed results |
| `TranscriptTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchTranscript(url)` and `extractChapters(transcript)`. Reads from `src/main/resources/sample-data/transcripts/*.json` for deterministic offline output. | called from TRANSCRIPT task | returns `Transcript` |
| `SummaryTools` | function-tools class | Implements `extractKeyPoints(transcript)` and `outlineSections(keyPoints)`. Pure in-memory transformations. | called from SUMMARISE task | returns `List<KeyPoint>` / `List<SectionOutline>` |
| `DraftTools` | function-tools class | Implements `writeSectionBody(outline, keyPoints)` and `composeIntroduction(keyPoints)`. | called from DRAFT task | returns `SectionBody` / `String` |
| `PolishTools` | function-tools class | Implements `polishProse(sectionBody)` and `writeConclusion(sections)`. | called from POLISH task | returns `String` / `String` |
| `PublishGuardrail` | `before-agent-response` guardrail (registered on `BlogAgent`) | Runs after the POLISH task's agent response but before `PostPolished` is recorded. Scans the response for prohibited content patterns and brand-safety violations. On reject, returns a structured error to the agent loop and records `GuardrailRejected`. On accept, the response is passed through unchanged. | POLISH task response | accept / structured-reject |
| `EditorialScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `BlogPost`, `BlogDraft`, `VideoSummary`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `BlogPostView` | `View` | Read model: one row per blog post for the UI. | `BlogPostEntity` events | `BlogEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Chapter(String title, int startSeconds, int endSeconds) {}

record Transcript(
    String videoUrl,
    String text,
    int durationSeconds,
    List<Chapter> chapters,
    Instant extractedAt
) {}

record KeyPoint(String pointId, String text, int sourceStartSeconds) {}

record SectionOutline(String sectionId, String heading, List<String> pointIds) {}

record VideoSummary(
    List<KeyPoint> keyPoints,
    List<SectionOutline> sections,
    Instant summarisedAt
) {}

record SectionBody(String sectionId, String heading, String body) {}

record BlogDraft(
    String introduction,
    List<SectionBody> sections,
    Instant draftedAt
) {}

record SourceTimestamp(String label, int startSeconds) {}

record PostSection(
    String sectionId,
    String heading,
    String body,
    List<SourceTimestamp> sourceTimestamps
) {}

record BlogPost(
    String title,
    String introduction,
    List<PostSection> sections,
    String conclusion,
    int wordCount,
    Instant polishedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record BlogPostRecord(
    String postId,
    Optional<String> videoUrl,
    Optional<Transcript> transcript,
    Optional<VideoSummary> summary,
    Optional<BlogDraft> draft,
    Optional<BlogPost> post,
    Optional<EvalResult> eval,
    BlogPostStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BlogPostStatus {
    CREATED, TRANSCRIBING, TRANSCRIBED, SUMMARISING, SUMMARISED,
    DRAFTING, DRAFTED, POLISHING, POLISHED, EVALUATED, FAILED
}
```

Events on `BlogPostEntity`: `PostCreated`, `TranscriptStarted`, `TranscriptExtracted`, `SummaryStarted`, `SummaryProduced`, `DraftStarted`, `DraftWritten`, `PolishStarted`, `PostPolished`, `EvaluationScored`, `GuardrailRejected`, `PostFailed`.

Every nullable lifecycle field on the `BlogPostRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/posts` — body `{ videoUrl }` → `{ postId }`.
- `GET /api/posts` — list all posts, newest-first.
- `GET /api/posts/{id}` — one post.
- `GET /api/posts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: YouTube to Blog</title>`.

The App UI tab is a two-column layout: a left rail with the live list of posts (status pill + video URL + age) and a right pane with the selected post's detail — video URL, transcript preview, summary key-points and section outline, draft bodies, polished post sections, eval score chip, and a guardrail-rejection log strip if any content rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (publish gate)**: `PublishGuardrail` is registered on `BlogAgent` and runs after the POLISH task produces its response but before the `PostPolished` event is recorded on the entity. The guardrail scans the response text for a configurable prohibited-content list (harmful-health-advice phrases, affiliate spam markers, deceptive-disclosure phrases) and checks that no competitor brand names appear in the body text. On reject, the guardrail returns a structured `content-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, reason}` event. The agent loop retries within its 3-iteration budget. If all 3 iterations are rejected, the workflow records `PostFailed` and the entity transitions to `FAILED`.
- **E1 — `on-decision-eval`**: runs immediately after `PostPolished` lands, as `evalStep` inside the workflow. `EditorialScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): section count parity between `BlogDraft` and `BlogPost` (draft sections = post sections), every `PostSection.heading` is non-empty, the post's `wordCount` is within the 400–1200 word target range, and every `PostSection.body` is non-empty. Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `BlogAgent` → `prompts/blog-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded URL `https://www.youtube.com/watch?v=seed-akka-agents`; within 90 s the post reaches `EVALUATED` with a non-empty transcript, ≥ 2 key points, ≥ 2 sections, and an eval score chip on the card.
2. **J2** — The agent's POLISH task produces a response containing a prohibited phrase (mock LLM path). `PublishGuardrail` rejects the response; a `GuardrailRejected` event lands on the entity; the agent retries; the post eventually completes with clean content. The UI's rejection-log strip shows the one rejected response.
3. **J3** — A post whose mock-LLM draft produces 3 sections but whose polish step produces 2 sections is scored ≤ 3 with a rationale naming the section-parity failure; the UI flags the card.
4. **J4** — Each task's instructions and tool calls are visible in the per-post trace (logged at `INFO`); the TRANSCRIPT task's log shows only TRANSCRIPT-tool calls, the SUMMARISE task's log shows only SUMMARISE-tool calls, and so on. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named video-to-blog-pipeline demonstrating the sequential-pipeline x
content-editorial cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact sequential-pipeline-content-editorial-video-to-blog-pipeline.
Java package io.akka.samples.youtubetoblog. Akka 3.6.0. HTTP port 9242.

Components to wire (exactly):

- 1 AutonomousAgent BlogAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/blog-agent.md>) and four .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  3)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  TRANSCRIPT, SUMMARISE, DRAFT, and POLISH tool sets are ALL registered on the agent.
  The before-agent-response guardrail (PublishGuardrail) is registered on the agent via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries within
  its 3-iteration budget for the POLISH task.

- 1 Workflow BlogPipelineWorkflow per postId with five steps:
  * transcriptStep — emits TranscriptStarted on the entity, then calls componentClient
    .forAutonomousAgent(BlogAgent.class, "agent-" + postId).runSingleTask(
      TaskDef.instructions("Video URL: " + videoUrl + "\nPhase: TRANSCRIPT\nUse fetchTranscript
      and extractChapters to produce a full Transcript.")
        .metadata("postId", postId)
        .metadata("phase", "TRANSCRIPT")
        .taskType(BlogTasks.EXTRACT_TRANSCRIPT)
    ). Reads forTask(taskId).result(EXTRACT_TRANSCRIPT) to get Transcript. Writes
    BlogPostEntity.recordTranscript(transcript). WorkflowSettings.stepTimeout 90s.
  * summaryStep — emits SummaryStarted, then runSingleTask with TaskDef.instructions
    (formatSummaryContext(transcript, videoUrl)) and metadata.phase = "SUMMARISE", taskType
    SUMMARISE_VIDEO. Writes BlogPostEntity.recordSummary(summary). stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(summary, transcript)) and metadata.phase = "DRAFT", taskType
    DRAFT_POST. Writes BlogPostEntity.recordDraft(draft). stepTimeout 60s.
  * polishStep — emits PolishStarted, then runSingleTask with TaskDef.instructions
    (formatPolishContext(draft, summary)) and metadata.phase = "POLISH", taskType
    POLISH_POST. The before-agent-response guardrail fires on the POLISH task's final
    response before the workflow reads it. Writes BlogPostEntity.recordPost(post). stepTimeout 60s.
  * evalStep — runs the deterministic EditorialScorer over (post, draft, summary)
    and writes BlogPostEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(BlogPipelineWorkflow::error). The error step writes
  PostFailed and ends.

- 1 EventSourcedEntity BlogPostEntity (one per postId). State BlogPostRecord{postId,
  videoUrl: Optional<String>, transcript: Optional<Transcript>, summary: Optional<VideoSummary>,
  draft: Optional<BlogDraft>, post: Optional<BlogPost>, eval: Optional<EvalResult>,
  status: BlogPostStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  BlogPostStatus enum: CREATED, TRANSCRIBING, TRANSCRIBED, SUMMARISING, SUMMARISED,
  DRAFTING, DRAFTED, POLISHING, POLISHED, EVALUATED, FAILED. Events:
  PostCreated{videoUrl}, TranscriptStarted, TranscriptExtracted{transcript},
  SummaryStarted, SummaryProduced{summary}, DraftStarted, DraftWritten{draft},
  PolishStarted, PostPolished{post}, EvaluationScored{eval},
  GuardrailRejected{phase, reason}, PostFailed{reason}.
  Commands: create, startTranscript, recordTranscript, startSummary, recordSummary,
  startDraft, recordDraft, startPolish, recordPost, recordEvaluation,
  recordGuardrailRejection, fail, getPost. emptyState() returns BlogPostRecord.initial("")
  with all Optional fields as Optional.empty() and no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 View BlogPostView with row type BlogPostRow that mirrors BlogPostRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes BlogPostEntity events. ONE
  query getAllPosts: SELECT * AS posts FROM blog_post_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * BlogEndpoint at /api with POST /posts (body {videoUrl}; mints postId; calls
    BlogPostEntity.create(videoUrl); then starts BlogPipelineWorkflow with id
    "pipeline-" + postId; returns {postId}), GET /posts (list from getAllPosts,
    sorted newest-first), GET /posts/{id} (one row), GET /posts/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- BlogTasks.java declaring four Task<R> constants:
    EXTRACT_TRANSCRIPT = Task.name("Extract transcript").description("Fetch the video
      transcript and identify chapter markers").resultConformsTo(Transcript.class);
    SUMMARISE_VIDEO = Task.name("Summarise video").description("Extract key points from
      the transcript and outline blog sections").resultConformsTo(VideoSummary.class);
    DRAFT_POST = Task.name("Draft post").description("Write section bodies from the
      section outline and key points").resultConformsTo(BlogDraft.class);
    POLISH_POST = Task.name("Polish post").description("Refine prose, add conclusion,
      finalise the blog post").resultConformsTo(BlogPost.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- TranscriptTools.java — @FunctionTool fetchTranscript(String videoUrl) -> Transcript
  reading from src/main/resources/sample-data/transcripts/*.json keyed by video slug;
  @FunctionTool extractChapters(String transcript) -> List<Chapter> parsing chapter markers.

- SummaryTools.java — @FunctionTool extractKeyPoints(Transcript transcript) -> List<KeyPoint>
  (one KeyPoint per chapter, with pointId minted as "p-" + sha1(chapter.title).substring(0,8));
  @FunctionTool outlineSections(List<KeyPoint> keyPoints) -> List<SectionOutline>
  (deterministic grouping — one SectionOutline per chapter, sectionId = "s-" + index).

- DraftTools.java — @FunctionTool writeSectionBody(SectionOutline outline,
  List<KeyPoint> keyPoints) -> SectionBody (heading from outline, body from key-point texts
  joined with linking phrases); @FunctionTool composeIntroduction(List<KeyPoint> keyPoints)
  -> String (2-3 sentence intro covering the top 3 key points).

- PolishTools.java — @FunctionTool polishProse(SectionBody sectionBody) -> String
  (returns improved prose string); @FunctionTool writeConclusion(List<PostSection> sections)
  -> String (2-3 sentence conclusion).

- PublishGuardrail.java — implements the before-agent-response hook. Reads the POLISH task
  response text, scans for entries from the prohibited-content list loaded from
  src/main/resources/config/prohibited-phrases.txt (one phrase per line), and checks that no
  line in the response matches any entry. On reject, returns Guardrail.reject(
  "content-violation: response contains prohibited phrase '<phrase>'"). On reject ALSO calls
  BlogPostEntity.recordGuardrailRejection(phase, reason) so the rejection is visible in the
  UI's rejection-log strip and in the audit log. On accept, passes the response unchanged.

- EditorialScorer.java — pure deterministic logic (no LLM). Inputs: BlogPost, BlogDraft,
  VideoSummary. Outputs: EvalResult with score and rationale. Four checks, one point per
  check satisfied, starting from a base of 1: section parity (post.sections.size() ==
  draft.sections.size()), heading completeness (every PostSection.heading is non-empty),
  word count (400 <= post.wordCount <= 1200), and body completeness (every PostSection.body
  is non-empty). Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9242 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/urls.jsonl with 5 seeded URL lines covering the three
  surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/transcripts/*.json — three files keyed by video slug, each
  carrying a Transcript with 3-5 chapters and a full text blob with deterministic content
  so TranscriptTools.fetchTranscript returns the same result across restarts.

- src/main/resources/config/prohibited-phrases.txt — 10-15 sample prohibited phrases
  (e.g. "this is not financial advice" masking actual financial advice, affiliate link
  patterns, misinformation markers) used by PublishGuardrail.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — content
  editorial domain with no jurisdiction-specific regulations pre-wired.

- risk-survey.yaml at the project root with data.data_classes.pii = false (video URLs and
  transcripts do not contain personal data in the typical editorial use case),
  decisions.authority_level = recommend-only (the blog post is editorial content subject to
  human review before publishing), oversight.human_in_loop = true (an editor reviews the
  post before it is published), operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "prohibited-content", "section-parity-drift",
  "evidence-thin-section", "word-count-out-of-range"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/blog-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: YouTube to Blog", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of post cards; right = selected-post detail with video URL header, transcript
  preview, summary key-points and section outline, draft bodies, polished post sections,
  eval-score chip, rejection-log strip). Browser title exactly:
  <title>Akka Sample: YouTube to Blog</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below).
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(postId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    extract-transcript.json — 5 Transcript entries, each with 3-4 Chapter items and a
      text blob. Each entry's tool_calls array contains 2 calls: fetchTranscript(videoUrl)
      + extractChapters(text).
    summarise-video.json — 5 VideoSummary entries paired one-to-one with the transcript
      entries, each with 3-4 SectionOutline items and matching KeyPoint items. tool_calls:
      extractKeyPoints + outlineSections.
    draft-post.json — 5 BlogDraft entries paired one-to-one. Each carries 3-4 SectionBody
      items. tool_calls: composeIntroduction + writeSectionBody (one per section).
    polish-post.json — 5 BlogPost entries paired one-to-one. Plus 1 deliberately
      PROHIBITED-CONTENT entry whose introduction contains a prohibited phrase from
      prohibited-phrases.txt — the guardrail rejects it, the mock then falls through to a
      clean polish response. The mock should select the prohibited-content entry on the
      FIRST iteration of every 3rd post (modulo seed) so J2 is reproducible.
      tool_calls: polishProse (one per section) + writeConclusion.
- A MockModelProvider.seedFor(postId) helper makes per-post selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BlogAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BlogTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (transcriptStep 90s, summaryStep 60s, draftStep 60s, polishStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on BlogPostRecord is Optional<T>. The view table
  updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: BlogTasks.java with EXTRACT_TRANSCRIPT, SUMMARISE_VIDEO, DRAFT_POST,
  POLISH_POST constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9242 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BlogAgent). The
  on-decision eval is rule-based (EditorialScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent;
  the before-agent-response guardrail (PublishGuardrail) moderates the POLISH task's
  final response. Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: transcriptStep writes Transcript onto
  the entity, summaryStep reads it and builds the SUMMARISE task's instruction context,
  draftStep reads VideoSummary, polishStep reads BlogDraft. The agent itself is stateless
  across phases.
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
