# Sample Specification — `content-pipeline`

The natural-language input for `/akka:specify`. The whole file — Sections 1–12 — is the spec; run `/akka:specify @SPEC.md` from inside the folder.

---

## 1. System name + pitch

**System name:** Content Pipeline.
**One-line pitch:** The user submits a topic; a research agent gathers sources, a writer agent drafts an article from the brief, a self-critique pass scores the draft, and the article publishes — each stage visible in the UI.

## 2. What this blueprint demonstrates

A sequential-pipeline coordination pattern in the content-editorial domain: one Workflow chains a research stage, a writing stage, a critique stage, and a publish stage, each stage handing a typed result to the next. The governance pattern wires two guardrails (a before-tool-call guard on the research agent's web-search tool that enforces an allowed-domain list, and a before-agent-response guard on the writer agent that runs a terminal style and safety check) plus a non-blocking on-decision eval (a self-critique pass that scores the draft before publishing).

## 3. User-facing flows

1. The user enters a topic in the App UI and submits. The system returns an `articleId` and the article appears in `RESEARCHING` state.
2. The research agent completes; the article moves to `WRITING` and the research summary and source list are shown.
3. The writer agent completes; the article moves to `CRITIQUING` and the draft title and body are shown.
4. The critique pass completes; a score and notes appear; the article moves to `PUBLISHED` with a published URL.
5. Without any UI interaction, the simulator seeds topics on a timer and the same pipeline runs for each.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ResearchAgent | AutonomousAgent | Topic → research brief; calls a simulated web-search tool guarded by an allowed-domain list | ContentPipelineWorkflow | ContentPipelineWorkflow |
| WriterAgent | AutonomousAgent | Research brief → article draft; terminal style/safety check before returning | ContentPipelineWorkflow | ContentPipelineWorkflow |
| CritiqueAgent | Agent | Draft → critique score + notes (self-critique eval) | ContentPipelineWorkflow | ArticleEntity |
| ContentPipelineWorkflow | Workflow | Chains research → write → critique → publish | TopicConsumer | ResearchAgent, WriterAgent, CritiqueAgent, ArticleEntity |
| ArticleEntity | EventSourcedEntity | Holds article lifecycle state and events | ContentPipelineWorkflow | ArticlesView |
| InboundTopicQueue | EventSourcedEntity | Records inbound topics as events | ContentEndpoint, TopicSimulator | TopicConsumer |
| ArticlesView | View | CQRS read model of articles for UI list and SSE | ArticleEntity | ContentEndpoint |
| TopicConsumer | Consumer | Starts a workflow per inbound topic | InboundTopicQueue | ContentPipelineWorkflow |
| TopicSimulator | TimedAction | Drips canned topics every 30s | sample-events JSONL | InboundTopicQueue |
| ContentEndpoint | HttpEndpoint | REST API, SSE, metadata serving | UI / clients | InboundTopicQueue, ArticlesView |
| AppEndpoint | HttpEndpoint | Serves the static UI | browser | static-resources |

Names matter. `/akka:specify` uses them verbatim.

## 5. Data model

See `reference/data-model.md`. Authoritative records:

- `Article(String id, Optional<String> topic, ArticleStatus status, Optional<Instant> researchedAt, Optional<String> researchSummary, Optional<String> researchSources, Optional<Instant> draftedAt, Optional<String> title, Optional<String> body, Optional<Instant> critiquedAt, Optional<Double> critiqueScore, Optional<String> critiqueNotes, Optional<Instant> publishedAt, Optional<String> publishedUrl, Optional<Instant> failedAt, Optional<String> failureReason)` — entity state and view row. Every nullable lifecycle field is `Optional<T>` (Lesson 6).
- `ArticleStatus` enum: `RESEARCHING, WRITING, CRITIQUING, PUBLISHED, FAILED`.
- Events: `ResearchCompleted`, `ArticleDrafted`, `ArticleCritiqued`, `ArticlePublished`, `ArticleFailed`.
- Typed agent results: `ResearchBrief(String summary, String sources)`, `ArticleDraft(String title, String body)`, `CritiqueResult(double score, String notes)`.

## 6. API contract

See `reference/api-contract.md` for payload schemas. Top-level surface:

```
POST /api/topics                     -> { articleId }
GET  /api/articles ?status=...       -> { articles: [Article, ...] }
GET  /api/articles/{articleId}       -> Article
GET  /api/articles/sse               -> Server-Sent Events of Article
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

A single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Content Pipeline</title>`. Five tabs:

1. **Overview** — eyebrow "Overview" + headline (sample type), then four cards: Try it / How it works / Components / API contract. No subtitle.
2. **Architecture** — the four mermaid diagrams from `PLAN.md`, with the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
3. **Risk Survey** — renders `risk-survey.yaml` in the `matrix-card` / `matrix-row` style; `TO_BE_COMPLETED_BY_DEPLOYER` values render muted.
4. **Eval Matrix** — renders `eval-matrix.yaml` in the same style; each control's label carries a colored mechanism pill.
5. **App UI** — submit a topic; live article list via SSE; per-article stage timeline showing research summary, draft, critique score, and published URL.

Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). No hidden zombie panels. See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Three controls:

- **G1** — before-tool-call guardrail on `ResearchAgent`: the simulated web-search tool only returns sources whose domain is on an allowed list; off-list URLs are blocked.
- **G2** — before-agent-response guardrail on `WriterAgent`: a terminal style and safety check runs before the draft is returned; failing drafts are blocked and the step fails over.
- **E1** — on-decision eval: `CritiqueAgent` produces a self-critique score on the draft; non-blocking, surfaced in the UI before publish.

## 9. Agent prompts

- `ResearchAgent` — `prompts/research-agent.md`: gather a concise, sourced brief on the topic.
- `WriterAgent` — `prompts/writer-agent.md`: write a publishable article from the brief.
- `CritiqueAgent` — `prompts/critique-agent.md`: score the draft and note weaknesses.

## 10. Acceptance

See `reference/user-journeys.md`. Passing means:

1. **Submit a topic, watch it publish** — POST a topic; within ~60s the article reaches `PUBLISHED` with a non-empty `body` and a `publishedUrl`.
2. **Research domain guard blocks an off-list source** — the before-tool-call guard rejects a source outside the allowed-domain list; the brief contains only allowed sources.
3. **Writer style guard blocks a bad draft** — a draft failing the style/safety check does not reach `CRITIQUING`; the article fails over with a reason.
4. **Critique score surfaces before publish** — the article shows a numeric `critiqueScore` and notes once `CRITIQUING` completes.

---

## 11. Implementation directives

The whole SPEC.md (Sections 1–12) is the input to `/akka:specify @SPEC.md`.

```
Create a sample named content-pipeline demonstrating the
sequential-pipeline x content-editorial cell. Runs out of the box (no external
services; the web-search source is modeled in-process over canned data).
Maven group io.akka.samples. Maven artifact content-pipeline. Java package
io.akka.samples.contentpipeline. Akka 3.6.0. HTTP port 9942.

Components to wire (exactly):
- 2 AutonomousAgents:
  - ResearchAgent — definition() with instructions from prompts/research-agent.md
    and capability(TaskAcceptance.of(RESEARCH).maxIterationsPerTask(3)). Calls a
    simulated web-search tool returning canned sources. A before-tool-call
    guardrail enforces an allowed-domain list (akka.io, doc.akka.io, wikipedia.org,
    and any domain listed in src/main/resources/allowed-domains.txt); off-list
    domains are blocked. Returns a typed ResearchBrief{summary, sources}.
  - WriterAgent — definition() with instructions from prompts/writer-agent.md and
    capability(TaskAcceptance.of(WRITE).maxIterationsPerTask(3)). A
    before-agent-response guardrail runs a terminal style/safety check (min length,
    profanity, topic adherence) before returning; failing drafts are blocked.
    Returns a typed ArticleDraft{title, body}.
- 1 Agent CritiqueAgent — method critique(ArticleDraft) returning
  Effect<CritiqueResult> via effects().systemMessage(...).userMessage(...)
  .responseAs(CritiqueResult.class).thenReply(). Instructions from
  prompts/critique-agent.md. Non-blocking.
- 1 Workflow ContentPipelineWorkflow with steps researchStep -> writeStep ->
  critiqueStep -> publishStep. researchStep calls
  forAutonomousAgent(ResearchAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...), writes ResearchCompleted to ArticleEntity.
  writeStep calls WriterAgent the same way, writes ArticleDrafted. critiqueStep
  calls forAgent().inSession(id).method(CritiqueAgent::critique).invoke(draft),
  writes ArticleCritiqued. publishStep synthesizes a published URL, writes
  ArticlePublished. Override settings() with stepTimeout(60s) on researchStep,
  writeStep, critiqueStep; defaultStepRecovery(maxRetries(2).failoverTo(
  ContentPipelineWorkflow::error)); error step writes ArticleFailed.
- 1 EventSourcedEntity ArticleEntity holding Article (id, topic Optional<String>,
  ArticleStatus enum, and 14 Optional lifecycle fields per reference/data-model.md).
  Events: ResearchCompleted, ArticleDrafted, ArticleCritiqued, ArticlePublished,
  ArticleFailed. Commands: recordResearch, recordDraft, recordCritique,
  recordPublication, markFailed, getArticle. emptyState() returns
  Article.initial("") with no commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundTopicQueue with one command enqueueTopic(topic)
  emitting TopicQueued.
- 1 View ArticlesView with row type Article, table updater consuming ArticleEntity
  events. ONE query: getAllArticles SELECT * AS articles FROM articles_view. No
  WHERE status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in callers.
- 1 Consumer TopicConsumer subscribed to InboundTopicQueue events; on each event
  starts a ContentPipelineWorkflow with a fresh UUID.
- 1 TimedAction TopicSimulator (every 30s, reads next line from
  src/main/resources/sample-events/topics.jsonl and calls
  InboundTopicQueue.enqueueTopic).
- 2 HttpEndpoints: ContentEndpoint at /api with topics (POST), articles list
  (filter client-side from getAllArticles), single article, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- ContentPipelineTasks.java declaring two Task<R> constants: RESEARCH
  (resultConformsTo ResearchBrief), WRITE (ArticleDraft).
- ResearchBrief(String summary, String sources),
  ArticleDraft(String title, String body),
  CritiqueResult(double score, String notes).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9942
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/allowed-domains.txt with a few allowed source domains.
- src/main/resources/sample-events/topics.jsonl with 8 canned topic lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1, G2, E1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor (Runs out of the box), how to run, the tabs, an ASCII
  architecture diagram, project layout, API contract, license. NO
  governance-mechanisms section. NO configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ResearchAgent -> ResearchBrief, WriterAgent -> ArticleDraft, CritiqueAgent ->
  CritiqueResult; see src/main/resources/mock-responses/{research-agent,
  writer-agent,critique-agent}.json with 4-6 entries each). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes captured key material.

Mock LLM provider per-agent shapes:
- research-agent.json: entries shaped {summary: "...", sources: "akka.io, wikipedia.org"}.
- writer-agent.json: entries shaped {title: "...", body: "4-6 paragraph article"}.
- critique-agent.json: entries shaped {score: 0.0-1.0, notes: "..."}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- AutonomousAgent never silently downgraded to Agent (Lesson 1). ResearchAgent and
  WriterAgent extend AutonomousAgent; CritiqueAgent extends Agent — by design.
- View cannot auto-index enum columns; filter status client-side (Lesson 2).
- emptyState() never calls commandContext() (Lesson 3).
- Workflow step timeouts set explicitly to 60s on agent-calling steps (Lesson 4).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- Optional<T> for every nullable lifecycle field on the Article row record (Lesson 6).
- AutonomousAgent requires a companion ContentPipelineTasks.java (Lesson 7).
- Verify model names against the provider's current lineup (Lesson 8).
- Run command is "/akka:build" (Claude Code slash command), never "mvn akka:run"
  (Lesson 9).
- Explicit dev-mode http-port 9942 in application.conf (Lesson 10).
- Never write "Akka SDK" in user-facing wording (Lesson 11/21); descriptive
  integration labels only, never T1/T2/T3/T4 or "deferred" (Lesson 13/22); never
  competitor brand names (Lesson 23).
- static-resources/index.html includes the mermaid CSS overrides and theme
  variables from Lesson 24.
- Tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index; no hidden zombie panels (Lesson 26).
- UI is a single self-contained static-resources/index.html — no npm (Lesson 17).
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and the `PLAN.md` in this folder.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9942/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the mock-provider option), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
