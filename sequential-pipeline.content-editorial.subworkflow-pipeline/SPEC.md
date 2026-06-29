# SPEC — content-pipeline-subworkflow-orchestrator

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Content Pipeline Subworkflow Orchestrator.
**One-line pitch:** A user submits a topic; one `ContentPipelineAgent` walks it through three task phases — **RESEARCH** source material, **DRAFT** an article, **REVIEW** the draft for quality — where each phase's tools delegate to dedicated sub-workflows, a `before-agent-response` guardrail checks tone and factuality on every agent response leaving the REVIEW task, and an editor-approval gate controls the publish transition.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain, with sub-workflows as the tool implementation layer. One `ContentPipelineAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RESEARCH task's typed `ResearchFindings` becomes the DRAFT task's instruction context; the DRAFT task's typed `Draft` and the upstream `ResearchFindings` become the REVIEW task's instruction context.

What makes this entry distinct from a plain sequential pipeline: each phase's tools do not call simple functions — they start Akka Workflows. `ResearchTools.gatherSources(topic)` starts a `ResearchSubworkflow` instance and waits for its result; `DraftTools.buildOutline(findings)` starts a `DraftSubworkflow` instance; `ReviewTools.checkFacts(draft, findings)` starts a `ReviewSubworkflow` instance. Each sub-workflow carries its own internal multi-step execution. The agent treats them as ordinary typed tool calls.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** intercepts every response leaving `ContentPipelineAgent` on the REVIEW task. It checks the completed `Draft` for off-brand language (a configurable phrase blocklist) and for unverified factual claims (every factual sentence must trace to a `SourceRef.url` present in the `ResearchFindings`). A response that fails either check is rejected before it returns to the workflow, and the agent retries within its 4-iteration budget.
- A **human-in-the-loop approval step** (`awaitApprovalStep`) pauses `ContentPipelineWorkflow` after `REVIEWED`. The article transitions to `AWAITING_APPROVAL`. An editor POST to `/api/articles/{id}/decision` with `{approved: true/false, note}` resumes the workflow, transitioning the article to `APPROVED → PUBLISHED` or `REJECTED`.

The blueprint shows that sub-workflows as tools are transparent to the pipeline's sequential-dependency contract: the agent's typed task results are still the mechanism that carries information between phases; sub-workflows are an implementation detail of how each phase's tools execute.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **topic** into the input (or picks one of three seeded topics — `AI regulation in financial services`, `Open-source LLM deployment guide`, `Synthetic data for model training`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/articles` and receives an `articleId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RESEARCHING` — the workflow has started `researchStep` and the agent has been handed the RESEARCH task.
4. Within ~20–30 s the card reaches `RESEARCHED` — a table of `SourceRef` entries appears in the card detail. The agent's RESEARCH task returned; the workflow recorded `ResearchCompleted` and ran the DRAFT task.
5. Within ~20–30 s more the card reaches `DRAFTED`. The `Draft` is visible — headline, lead paragraph, section list.
6. Within ~20–30 s more the card reaches `REVIEWED`. The `ReviewResult` is visible (review notes, score 1–5). Within 1 s of that, `AWAITING_APPROVAL` is set — the workflow has reached `awaitApprovalStep`.
7. An editor (or the user playing editor) clicks **Approve** or **Reject** in the right pane. If approved, the card transitions to `APPROVED` then `PUBLISHED`. If rejected, the card shows `REJECTED` with the editor's note.
8. The user can submit another topic; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ContentEndpoint` | `HttpEndpoint` | `/api/articles/*` — submit, list, get, SSE, editor decision; serves `/api/metadata/*`. | — | `ArticleEntity`, `ArticleView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ArticleEntity` | `EventSourcedEntity` | Per-article lifecycle: created → researching → researched → drafting → drafted → reviewing → reviewed → awaiting_approval → approved → published / rejected / failed. Source of truth. | `ContentEndpoint`, `ContentPipelineWorkflow` | `ArticleView` |
| `ContentPipelineWorkflow` | `Workflow` | One workflow per article. Steps: `researchStep` → `draftStep` → `reviewStep` → `guardStep` → `awaitApprovalStep` → `publishStep` (or `rejectStep`). Each agent-calling step runs one task on the agent, reads the typed result, writes the corresponding event onto `ArticleEntity`, then advances. `guardStep` runs `ToneGuardrail` synchronously. `awaitApprovalStep` pauses until an editor decision arrives. | started by `ContentEndpoint` after `CREATED` | `ContentPipelineAgent`, `ArticleEntity` |
| `ContentPipelineAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `ContentPipelineTasks.java`: `RESEARCH_TOPIC` → `ResearchFindings`, `DRAFT_CONTENT` → `Draft`, `REVIEW_DRAFT` → `ReviewResult`. | invoked by `ContentPipelineWorkflow` | returns typed results |
| `ResearchSubworkflow` | `Workflow` | Executes research for one article. Steps: `gatherSourcesStep` (reads from `sample-data/sources/`) → `extractFactsStep` (derives key facts from source excerpts) → `summarizeStep` (produces `ResearchFindings.summary`). Started by `ResearchTools`; its result is returned to the agent. | invoked by `ResearchTools` | returns `ResearchFindings` |
| `DraftSubworkflow` | `Workflow` | Executes drafting for one article. Steps: `buildOutlineStep` (turns `ResearchFindings` into `Outline`) → `writeSectionsStep` (produces `List<DraftSection>`) → `assembleStep` (combines into `Draft`). Started by `DraftTools`. | invoked by `DraftTools` | returns `Draft` |
| `ReviewSubworkflow` | `Workflow` | Executes review for one article. Steps: `factCheckStep` (cross-references `Draft.sections[].citedUrls` against `ResearchFindings.sources`) → `toneScoreStep` (deterministic phrase-match) → `structureScoreStep` (section parity, lead paragraph presence). Started by `ReviewTools`. | invoked by `ReviewTools` | returns `ReviewResult` |
| `ResearchTools` | function-tools class (POJO with `@FunctionTool` methods) | `gatherSources(topic)` starts `ResearchSubworkflow` and waits for `ResearchFindings`. RESEARCH-phase only. | called from RESEARCH task | returns `ResearchFindings` |
| `DraftTools` | function-tools class | `buildOutline(findings)` starts `DraftSubworkflow` and waits for `Draft`. DRAFT-phase only. | called from DRAFT task | returns `Draft` |
| `ReviewTools` | function-tools class | `checkFacts(draft, findings)` starts `ReviewSubworkflow` and waits for `ReviewResult`. REVIEW-phase only. | called from REVIEW task | returns `ReviewResult` |
| `ToneGuardrail` | `before-agent-response` guardrail (registered on `ContentPipelineAgent`) | Runs after the REVIEW task returns, before the typed result lands in the workflow. Checks the `Draft` embedded in the agent's response for off-brand phrases and unverified factual claims. Rejects and triggers retry if either check fails. | every agent response on REVIEW task | accept / structured-reject |
| `ArticleView` | `View` | Read model: one row per article for the UI. | `ArticleEntity` events | `ContentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SourceRef(String title, String url, String excerpt, Instant retrievedAt) {}

record ResearchFindings(
    List<SourceRef> sources,
    String summary,
    Instant researchedAt
) {}

record Outline(List<String> sectionHeadings, String thesis, Instant outlinedAt) {}

record DraftSection(String heading, String body, List<String> citedUrls) {}

record Draft(
    String headline,
    String leadParagraph,
    List<DraftSection> sections,
    Outline outline,
    Instant draftedAt
) {}

record ReviewNote(String category, String description, String severity) {}
// category: "factuality" | "tone" | "structure"
// severity: "error" | "warning" | "info"

record ReviewResult(
    List<ReviewNote> notes,
    int score,            // 1..5
    Instant reviewedAt
) {}

record GuardResult(
    boolean passed,
    Optional<String> issue,
    Instant checkedAt
) {}

record EditorDecision(
    String articleId,
    boolean approved,
    Optional<String> note,
    Instant decidedAt
) {}

record ArticleRecord(
    String articleId,
    Optional<String> topic,
    Optional<ResearchFindings> research,
    Optional<Draft> draft,
    Optional<ReviewResult> review,
    Optional<GuardResult> guardResult,
    Optional<EditorDecision> editorDecision,
    ArticleStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<GuardrailRejection> guardrailRejections
) {}

enum ArticleStatus {
    CREATED, RESEARCHING, RESEARCHED, DRAFTING, DRAFTED,
    REVIEWING, REVIEWED, AWAITING_APPROVAL,
    APPROVED, PUBLISHED, REJECTED, FAILED
}
```

Events on `ArticleEntity`: `ArticleCreated`, `ResearchStarted`, `ResearchCompleted`, `DraftStarted`, `DraftCompleted`, `ReviewStarted`, `ReviewCompleted`, `GuardChecked`, `ApprovalRequested`, `EditorApproved`, `EditorRejected`, `ArticlePublished`, `GuardrailRejected`, `ArticleFailed`.

Every nullable lifecycle field on the `ArticleRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/articles` — body `{ topic }` → `{ articleId }`.
- `GET /api/articles` — list all articles, newest-first.
- `GET /api/articles/{id}` — one article.
- `GET /api/articles/sse` — Server-Sent Events; one event per state transition.
- `POST /api/articles/{id}/decision` — body `{ approved, note }` → `{ articleId, status }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Content Pipeline Subworkflow Orchestrator</title>`.

The App UI tab is a two-column layout: a left rail with the live list of articles (status pill + topic + age + guardrail-flag dot) and a right pane with the selected article's detail — topic, research sources table, draft preview (headline + lead + sections), review notes list, eval score chip, editor decision panel (Approve / Reject buttons when status is `AWAITING_APPROVAL`), and a guardrail-rejection log strip if any tone/factuality rejections fired.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (tone + factuality gate)**: `ToneGuardrail` is registered on `ContentPipelineAgent` and runs before every agent response on the REVIEW task. It reads the `Draft` embedded in the typed `ReviewResult` and applies two checks: (1) **tone check** — the `Draft.leadParagraph` and each `DraftSection.body` are scanned for a configurable phrase blocklist (loaded from `src/main/resources/config/blocked-phrases.txt`); any match is a tone violation. (2) **factuality check** — every `DraftSection.citedUrls[i]` must appear in `ResearchFindings.sources[].url`; a URL that does not appear in the recorded sources is an unverified claim. Either failure causes the guardrail to return a structured `tone-violation` or `factuality-violation` error to the agent loop. The agent retries within its 4-iteration budget; if all iterations fail, the workflow transitions the article to `FAILED`. On guardrail rejection, the guardrail also calls `ArticleEntity.recordGuardrailRejection(phase, reason)` so the rejection is visible in the UI's rejection-log strip.
- **H1 — human-in-the-loop editor approval**: after `ReviewCompleted` and `GuardChecked` both land, `ContentPipelineWorkflow.awaitApprovalStep` transitions the article to `AWAITING_APPROVAL` and pauses. An editor POSTs to `POST /api/articles/{id}/decision` with `{ approved: true/false, note }`. The workflow resumes: on `approved = true` it emits `EditorApproved` → `ArticlePublished` and transitions to `PUBLISHED`; on `approved = false` it emits `EditorRejected` and transitions to `REJECTED`. No publish event can land without an explicit editor decision — the `AWAITING_APPROVAL` pause is the hard gate.

## 9. Agent prompts

- `ContentPipelineAgent` → `prompts/content-pipeline-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools from the matching phase's sub-workflow tools, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded topic `Open-source LLM deployment guide`; within 90 s the article reaches `AWAITING_APPROVAL` with non-empty research sources, ≥ 2 draft sections, and a review score chip on the card. An editor clicks Approve; the card reaches `PUBLISHED`.
2. **J2** — The agent's REVIEW task response contains a phrase from `blocked-phrases.txt` in the mock LLM's pre-loaded trajectory. `ToneGuardrail` rejects it; a `GuardrailRejected` event lands on the entity; the agent retries with a clean draft; the article eventually reaches `AWAITING_APPROVAL`.
3. **J3** — An editor submits a Reject decision with a note; the article transitions to `REJECTED`; the UI shows the rejection note; the workflow terminates cleanly.
4. **J4** — Each task's tool calls are limited to that phase's sub-workflow: RESEARCH task only starts `ResearchSubworkflow`, DRAFT task only starts `DraftSubworkflow`, REVIEW task only starts `ReviewSubworkflow`. The service log (at DEBUG) confirms no cross-phase sub-workflow is started.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sequential-pipeline-content-editorial-subworkflow-pipeline demonstrating
the sequential-pipeline x content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact
sequential-pipeline-content-editorial-subworkflow-pipeline.
Java package io.akka.samples.contentpipelinesubworkfloworchestrator. Akka 3.6.0. HTTP port 9993.

Components to wire (exactly):

- 1 AutonomousAgent ContentPipelineAgent — extends
  akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/content-pipeline-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(ResearchTools.class, DraftTools.class, ReviewTools.class) — ALL three tool
  classes are registered; phase isolation is the job of the per-phase sub-workflow routing,
  not conditional .tools(...) wiring. The before-agent-response guardrail (ToneGuardrail)
  is registered on the agent via the agent's guardrail-configuration block.

- 1 Workflow ContentPipelineWorkflow per articleId with six steps:
  * researchStep — emits ResearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(ContentPipelineAgent.class, "agent-" + articleId).runSingleTask(
      TaskDef.instructions("Topic: " + topic + "\nPhase: RESEARCH\nCall gatherSources to
      retrieve source material for this topic.")
        .metadata("articleId", articleId)
        .metadata("phase", "RESEARCH")
        .taskType(ContentPipelineTasks.RESEARCH_TOPIC)
    ). Reads result to get ResearchFindings. Writes ArticleEntity.recordResearch(findings).
    WorkflowSettings.stepTimeout 90s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions(
    formatDraftContext(findings, topic)) and metadata.phase = "DRAFT", taskType
    DRAFT_CONTENT. Writes ArticleEntity.recordDraft(draft). stepTimeout 90s.
  * reviewStep — emits ReviewStarted, then runSingleTask with TaskDef.instructions(
    formatReviewContext(draft, findings, topic)) and metadata.phase = "REVIEW", taskType
    REVIEW_DRAFT. Writes ArticleEntity.recordReview(reviewResult). stepTimeout 90s.
  * guardStep — runs ToneGuardrail.check(draft, findings) synchronously (ToneGuardrail
    is also registered as the agent's before-agent-response hook, but guardStep also
    runs it as an explicit post-review gate to write GuardChecked onto the entity for
    audit visibility). Writes ArticleEntity.recordGuardResult(guardResult). stepTimeout 10s.
  * awaitApprovalStep — emits ApprovalRequested on the entity (transitioning to
    AWAITING_APPROVAL). This step has no timeout — it pauses until resumed by an external
    call. ContentPipelineWorkflow exposes a resume(EditorDecision) method that the
    ContentEndpoint calls when an editor POSTs to /api/articles/{id}/decision.
  * publishStep (on approved=true) — emits EditorApproved then ArticlePublished.
    stepTimeout 5s.
  * rejectStep (on approved=false) — emits EditorRejected. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(ContentPipelineWorkflow::error). The error step writes
  ArticleFailed and ends.

- 1 EventSourcedEntity ArticleEntity (one per articleId). State ArticleRecord{articleId,
  topic: Optional<String>, research: Optional<ResearchFindings>, draft: Optional<Draft>,
  review: Optional<ReviewResult>, guardResult: Optional<GuardResult>,
  editorDecision: Optional<EditorDecision>, status: ArticleStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, guardrailRejections: List<GuardrailRejection>}.
  ArticleStatus enum: CREATED, RESEARCHING, RESEARCHED, DRAFTING, DRAFTED, REVIEWING,
  REVIEWED, AWAITING_APPROVAL, APPROVED, PUBLISHED, REJECTED, FAILED.
  Events: ArticleCreated{topic}, ResearchStarted, ResearchCompleted{findings},
  DraftStarted, DraftCompleted{draft}, ReviewStarted, ReviewCompleted{reviewResult},
  GuardChecked{guardResult}, ApprovalRequested, EditorApproved{decision},
  EditorRejected{decision}, ArticlePublished, GuardrailRejected{phase, reason},
  ArticleFailed{reason}.
  Commands: create, startResearch, recordResearch, startDraft, recordDraft, startReview,
  recordReview, recordGuardResult, requestApproval, recordEditorDecision, publish,
  recordGuardrailRejection, fail, getArticle.
  emptyState() returns ArticleRecord.initial("") with all Optional fields as
  Optional.empty(), guardrailRejections = List.of(), and status = CREATED. No
  commandContext() reference in emptyState() (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View ArticleView with row type ArticleRow that mirrors ArticleRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes ArticleEntity events.
  ONE query getAllArticles: SELECT * AS articles FROM article_view. No WHERE status filter
  — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ContentEndpoint at /api with POST /articles (body {topic}; mints articleId; calls
    ArticleEntity.create(topic); then starts ContentPipelineWorkflow with id
    "pipeline-" + articleId; returns {articleId}), GET /articles (list from
    getAllArticles, sorted newest-first), GET /articles/{id} (one row),
    GET /articles/sse (Server-Sent Events forwarded from the view's stream-updates),
    POST /articles/{id}/decision (body {approved, note}; calls
    ContentPipelineWorkflow.resume(EditorDecision); returns {articleId, status}),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

- 3 sub-Workflows (each is a standard Akka Workflow, separate from ContentPipelineWorkflow):
  * ResearchSubworkflow — id keyed by "research-" + articleId. Steps:
    gatherSourcesStep: reads src/main/resources/sample-data/sources/<topic-slug>.json,
      returns List<SourceRef>.
    extractFactsStep: for each SourceRef, extracts 1-2 key facts as strings.
    summarizeStep: joins facts into a paragraph (deterministic template join) to produce
      ResearchFindings.summary.
    Returns ResearchFindings{sources, summary, researchedAt}. stepTimeout 15s each.
  * DraftSubworkflow — id keyed by "draft-" + articleId. Steps:
    buildOutlineStep: from ResearchFindings, produces Outline{sectionHeadings, thesis}.
    writeSectionsStep: for each heading, produces DraftSection{heading, body, citedUrls}
      where body is composed from the matching source excerpts and citedUrls contains
      only urls present in ResearchFindings.sources.
    assembleStep: assembles Draft{headline, leadParagraph, sections, outline}.
    Returns Draft. stepTimeout 15s each.
  * ReviewSubworkflow — id keyed by "review-" + articleId. Steps:
    factCheckStep: for each DraftSection.citedUrl, verifies it appears in
      ResearchFindings.sources[].url. Produces ReviewNote{category:"factuality",...}
      for each failing citation.
    toneScoreStep: scans headline, leadParagraph, section bodies against
      blocked-phrases.txt. Produces ReviewNote{category:"tone",...} for each match.
    structureScoreStep: checks section count >= 2 and leadParagraph.length() > 0.
      Produces ReviewNote{category:"structure",...} if either fails.
    Returns ReviewResult{notes, score, reviewedAt}. score = 5 - min(notes.size(), 4),
    base 1. stepTimeout 10s each.

Companion classes:

- ContentPipelineTasks.java declaring three Task<R> constants:
    RESEARCH_TOPIC = Task.name("Research topic").description("Gather source material
      for a topic by calling gatherSources").resultConformsTo(ResearchFindings.class);
    DRAFT_CONTENT = Task.name("Draft content").description("Produce a structured article
      draft from research findings by calling buildOutline").resultConformsTo(Draft.class);
    REVIEW_DRAFT = Task.name("Review draft").description("Review the draft for factuality,
      tone, and structure by calling checkFacts").resultConformsTo(ReviewResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {RESEARCH, DRAFT, REVIEW}. Each function-tool method carries its phase
  constant so the before-agent-response guardrail and logs can identify phase context.

- ResearchTools.java — @FunctionTool gatherSources(String topic) -> ResearchFindings,
  implemented by starting ResearchSubworkflow("research-" + articleId) via componentClient
  and calling .getResult() on the completed workflow.

- DraftTools.java — @FunctionTool buildOutline(ResearchFindings findings) -> Draft,
  implemented by starting DraftSubworkflow("draft-" + articleId) via componentClient
  with the ResearchFindings payload and calling .getResult().

- ReviewTools.java — @FunctionTool checkFacts(Draft draft, ResearchFindings findings) ->
  ReviewResult, implemented by starting ReviewSubworkflow("review-" + articleId) with
  both payloads and calling .getResult().

- ToneGuardrail.java — implements the before-agent-response hook. On every REVIEW task
  response, reads the Draft in the agent's typed result, applies the two-check matrix
  (blocked phrases against draft text, cited URLs against ResearchFindings.sources). On
  failure, calls ArticleEntity.recordGuardrailRejection(phase, reason) and returns
  Guardrail.reject("tone-violation: <phrase>" or "factuality-violation: <url> not in
  sources"). On accept, passes.

- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9993 and the three model-provider blocks
  (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash)
  reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/topics.jsonl with 5 seeded topic lines covering the
  three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/sources/*.json — three files keyed by seeded topic,
  each carrying 5-8 SourceRef entries with deterministic content.

- src/main/resources/config/blocked-phrases.txt — one phrase per line; used by
  ToneGuardrail and ReviewSubworkflow.toneScoreStep.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms
  in Section 8. Matching simplified_view list.

- risk-survey.yaml at the project root reflecting content-editorial domain decisions.

- prompts/content-pipeline-agent.md loaded as the agent system prompt.

- README.md at the project root. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of article cards; right = selected-article detail with topic header,
  research sources table, draft preview, review notes, eval-score chip, editor decision
  panel, rejection-log strip). Browser title exactly:
  <title>Akka Sample: Content Pipeline Subworkflow Orchestrator</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ContentPipelineAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion
  ContentPipelineTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent or sub-workflow has an explicit
  stepTimeout (researchStep 90s, draftStep 90s, reviewStep 90s, guardStep 10s,
  awaitApprovalStep no timeout — it waits for external resume, publishStep 5s,
  rejectStep 5s, error 5s). Each sub-workflow step 15s (ResearchSubworkflow,
  DraftSubworkflow) or 10s (ReviewSubworkflow).
- Lesson 5: WorkflowSettings is the nested Workflow.WorkflowSettings — do not import a
  top-level WorkflowSettings class.
- Lesson 6: every nullable lifecycle field on ArticleRecord is Optional<T>.
- Lesson 7: ContentPipelineTasks.java with RESEARCH_TOPIC, DRAFT_CONTENT, REVIEW_DRAFT
  constants is mandatory.
- Lesson 8: model names — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9993 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in prose.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides and
  themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements — no more.
- Single-agent invariant: exactly ONE AutonomousAgent (ContentPipelineAgent). Sub-workflows
  (ResearchSubworkflow, DraftSubworkflow, ReviewSubworkflow) are Workflows, not agents.
- Sequential-pipeline invariant: each phase's typed task result is the only path
  information travels to the next phase. Sub-workflows are an implementation detail of
  how the tool executes; the agent is stateless across phases.
- HITL invariant: no ArticlePublished event can land without a prior EditorApproved event
  on the same articleId. awaitApprovalStep must be the hard gate.
- The Overview tab's Try-it card shows just "/akka:build". Per Lesson 25, /akka:specify
  handles the key during generation.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
