# SPEC — social-post-team

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 are the input; Section 12 chains the rest of the workflow.

---

## 1. System name + pitch

**System name:** Social Post Team.
**One-line pitch:** A marketer types a creative concept into the App UI; a pipeline of agents researches context, writes Instagram caption copy and visual direction, runs a brand and platform-policy check, and pauses for the marketer to approve before the post is marked published.

## 2. What this blueprint demonstrates

The coordination pattern is a sequential pipeline: a workflow drives a fixed sequence of agent steps, each consuming the prior step's typed output. The governance pattern wires two guardrails and one human gate — a before-tool-call guardrail allowlists any URL the research agent fetches, a before-agent-response guardrail checks brand voice and Instagram platform policy before any caption reaches a human, and a human approval gate holds the post until a marketer signs off.

## 3. User-facing flows

1. The marketer submits a concept (one line of creative direction). The system creates a post in `RESEARCHING` and returns its id.
2. The research agent gathers context via a web-search tool and a page-browse tool; the post moves to `COMPOSING`.
3. The copy agent writes a caption with hashtags; the visual-director agent writes a visual brief. The before-agent-response guardrail checks both; the post moves to `AWAITING_APPROVAL`.
4. The marketer reads the caption and visual brief in the App UI and clicks Approve or Reject. Approve drives the post to `APPROVED` then `PUBLISHED` with a permalink. Reject records a reason and ends in `REJECTED`.
5. A post left in `AWAITING_APPROVAL` past the threshold transitions to `ESCALATED` automatically.

These map to `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ResearchAgent | Agent | Gather context for the concept via web-search + browse tools | PostWorkflow | PostWorkflow |
| CopyAgent | Agent | Write Instagram caption + hashtags from concept and research | PostWorkflow | PostWorkflow |
| VisualDirectorAgent | Agent | Write a visual brief (composition, mood, color) | PostWorkflow | PostWorkflow |
| PostWorkflow | Workflow | Orchestrate research → compose → brand check → await approval → publish | ConceptConsumer | PostEntity |
| PostEntity | EventSourcedEntity | Hold the post brief and its lifecycle | PostWorkflow, PostEndpoint | PostsView |
| ConceptQueue | EventSourcedEntity | Inbound queue of submitted concepts | PostEndpoint, ConceptSimulator | ConceptConsumer |
| PostsView | View | CQRS read model of all posts | PostEntity | PostEndpoint |
| ConceptConsumer | Consumer | Start one PostWorkflow per queued concept | ConceptQueue | PostWorkflow |
| ConceptSimulator | TimedAction | Drip canned concepts from a JSONL file | — | ConceptQueue |
| StalePostMonitor | TimedAction | Escalate posts awaiting approval too long | PostsView | PostEntity |
| WebTools | HttpEndpoint | In-process web-search + page-browse tools the research agent calls | ResearchAgent | — |
| PostEndpoint | HttpEndpoint | REST + SSE + metadata | UI | ConceptQueue, PostEntity, PostsView |
| AppEndpoint | HttpEndpoint | Serve the static UI | browser | — |

## 5. Data model

Authoritative — `/akka:implement` writes records exactly as specified. See `reference/data-model.md`.

`PostBrief` record fields: `id` String, `concept` String, `status` PostStatus enum, and these lifecycle fields, every one `Optional<T>` (Lesson 6): `researchedAt` Instant, `researchNotes` String, `composedAt` Instant, `caption` String, `hashtags` List<String> (empty list when absent, never null), `visualBrief` String, `brandCheckPassed` Boolean, `brandCheckNotes` String, `approvedAt` Instant, `approvedBy` String, `approverComment` String, `rejectedAt` Instant, `rejectedBy` String, `rejectReason` String, `escalatedAt` Instant, `publishedAt` Instant, `publishedPermalink` String.

`PostStatus` enum: `RESEARCHING`, `COMPOSING`, `AWAITING_APPROVAL`, `APPROVED`, `REJECTED`, `ESCALATED`, `PUBLISHED`.

Events: `ConceptQueued`, `PostResearched`, `PostComposed`, `BrandChecked`, `PostApproved`, `PostRejected`, `PostEscalated`, `PostPublished`.

Typed agent results: `ResearchNotes(String summary, List<String> sources)`, `CaptionDraft(String caption, List<String> hashtags)`, `VisualBrief(String brief)`.

## 6. API contract

Inline surface (full schemas in `reference/api-contract.md`):

```
POST /api/concepts                 -> { postId }
POST /api/posts/{postId}/approve   -> 200 | 404
POST /api/posts/{postId}/reject    -> 200 | 404
GET  /api/posts                    -> { posts: [PostBrief, ...] }
GET  /api/posts/{postId}           -> PostBrief
GET  /api/posts/sse                -> Server-Sent Events of PostBrief
POST /api/tools/search             -> { results: [...] }   (in-process web-search tool)
POST /api/tools/browse             -> { text }             (in-process page-browse tool)
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Social Post Team</title>`. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, and removed tabs are deleted from the DOM rather than hidden (Lesson 26). Architecture tab renders the mermaid diagrams with the theme variables and CSS label overrides from Lesson 24. App UI tab: submit a concept, live SSE list of posts, Approve/Reject buttons when status is `AWAITING_APPROVAL`. See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1 — before-tool-call guardrail:** before ResearchAgent calls the web-search or browse tool, an allowlist guard rejects any URL outside the approved host list.
- **G2 — before-agent-response guardrail:** before CopyAgent and VisualDirectorAgent results are persisted, a brand-voice and Instagram platform-policy check runs; failing content is blocked from reaching the marketer.
- **H1 — hitl application gate:** PostWorkflow pauses in the await-approval step until a human approves or rejects.
- **P1 — eval-periodic:** StalePostMonitor escalates posts awaiting approval past the threshold.
- **A1 — ci-gate:** tests must pass before deploy.

## 9. Agent prompts

- `prompts/research-agent.md` — gather factual context and references for the concept using the search and browse tools.
- `prompts/copy-agent.md` — write an on-brand Instagram caption with hashtags.
- `prompts/visual-director-agent.md` — write a visual brief an art director or image model can execute.

## 10. Acceptance

Inline journeys (full set in `reference/user-journeys.md`):

1. **Submit a concept and watch it compose.** POST `/api/concepts`; the post appears via SSE, moves through `RESEARCHING` and `COMPOSING`, and lands in `AWAITING_APPROVAL` with a non-empty caption and visual brief.
2. **Approve and publish.** Approve drives the post to `PUBLISHED` with a non-null `publishedPermalink`.
3. **Reject.** Reject records the reason and ends in `REJECTED`.
4. **Escalate.** A post left in `AWAITING_APPROVAL` past the threshold transitions to `ESCALATED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named social-post-team demonstrating the
sequential-pipeline x sales-marketing cell. Runs out of the box (no external
services; the search and browse tools are in-process Akka endpoints).
Maven group io.akka.samples. Maven artifact social-post-team. Java package
io.akka.samples.socialpostteam. Akka 3.6.0. HTTP port 9273.

Components to wire (exactly):
- 3 Agents (request/response, extends akka.javasdk.agent.Agent):
  - ResearchAgent: method research(String concept) returning Effect<ResearchNotes>.
    Calls the web-search and browse tools (POST /api/tools/search, /api/tools/browse
    on the same service) via an Akka HTTP client. Uses
    effects().systemMessage(prompt).userMessage(...).responseAs(ResearchNotes.class).thenReply().
  - CopyAgent: method compose(ComposeInput) returning Effect<CaptionDraft>.
  - VisualDirectorAgent: method direct(ComposeInput) returning Effect<VisualBrief>.
  Each loads its system prompt from prompts/<agent>.md as a classpath resource.
- 1 Workflow PostWorkflow with steps: researchStep -> composeStep ->
  brandCheckStep -> awaitApprovalStep -> publishStep. researchStep calls
  forAgent(ResearchAgent.class).inSession(postId).method(ResearchAgent::research)
  .invoke(concept), writes PostResearched. composeStep calls CopyAgent and
  VisualDirectorAgent, writes PostComposed. brandCheckStep applies the
  before-agent-response guardrail; on pass writes BrandChecked and transitions
  to awaitApprovalStep; on fail ends. awaitApprovalStep polls PostEntity.getPost;
  on AWAITING_APPROVAL self-schedules a 5-second resume timer; on APPROVED
  transitions to publishStep; on REJECTED or ESCALATED ends. publishStep writes
  PostPublished with a generated permalink. Override settings() with
  stepTimeout(60s) on researchStep, composeStep, brandCheckStep, publishStep and
  defaultStepRecovery(maxRetries(2).failoverTo(PostWorkflow::error)).
- 1 EventSourcedEntity PostEntity holding a PostBrief record with id, concept,
  PostStatus enum, and the 17 Optional lifecycle fields listed in Section 5
  (hashtags is List<String> defaulting to empty, never null). Events:
  PostResearched, PostComposed, BrandChecked, PostApproved, PostRejected,
  PostEscalated, PostPublished. Commands: recordResearch, recordCompose,
  recordBrandCheck, approve, reject, markEscalated, recordPublication, getPost.
  emptyState() returns PostBrief.initial("", "") with placeholder identity and no
  commandContext() reference.
- 1 EventSourcedEntity ConceptQueue with a single command enqueue(concept)
  emitting ConceptQueued.
- 1 View PostsView with row type PostBrief, table updater consuming PostEntity
  events. ONE query: getAllPosts SELECT * AS posts FROM posts_view. No WHERE
  status filter (Akka cannot auto-index enum columns) — filter client-side in
  callers.
- 1 Consumer ConceptConsumer subscribed to ConceptQueue events; on each event
  starts a PostWorkflow with a fresh UUID.
- 2 TimedActions: ConceptSimulator (every 30s, reads next line from
  src/main/resources/sample-events/concepts.jsonl and calls ConceptQueue.enqueue);
  StalePostMonitor (every 30s, queries PostsView.getAllPosts, filters
  AWAITING_APPROVAL with composedAt > 2 min ago, calls PostEntity.markEscalated).
- 1 HttpEndpoint WebTools at /api/tools: POST /search returns canned results from
  src/main/resources/sample-data/search-results.json; POST /browse returns canned
  page text from src/main/resources/sample-data/pages/. Both apply the G1 URL
  allowlist guard.
- 1 HttpEndpoint PostEndpoint at /api: concepts, approve, reject, posts list
  (filter client-side), single post, SSE stream, and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/.
- 1 HttpEndpoint AppEndpoint: / -> 302 /app/index.html and /app/* ->
  static-resources/*.

Companion files:
- Records: ResearchNotes(String summary, List<String> sources),
  CaptionDraft(String caption, List<String> hashtags), VisualBrief(String brief),
  ComposeInput(String concept, String researchSummary),
  ApprovalDecision(String approvedBy, String comment),
  RejectDecision(String rejectedBy, String reason).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9273
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/concepts.jsonl with 8 canned concept lines.
- src/main/resources/sample-data/search-results.json and pages/ with canned
  tool outputs.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1, G2, H1, P1, A1 and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking deployer fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): elevator pitch, what you get,
  customise, what gets validated, license. No governance-mechanisms section, no
  configuration section.
- src/main/resources/static-resources/index.html — single self-contained HTML
  (no ui/ folder, no npm). Inline CSS + JS; runtime CDN imports for markdown and
  YAML are acceptable. Five tabs: Overview, Architecture, Risk Survey, Eval
  Matrix, App UI. Match the governance.html visual style (dark / yellow accent /
  Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for ANTHROPIC_API_KEY
  / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ResearchAgent -> ResearchNotes, CopyAgent -> CaptionDraft, VisualDirectorAgent
  -> VisualBrief; write src/main/resources/mock-responses/{research-agent,
  copy-agent,visual-director-agent}.json with 4-6 entries each). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider per-agent shapes (option a):
- research-agent.json: ResearchNotes with a 2-3 sentence summary and 2-3 fake
  source URLs from the allowlisted hosts.
- copy-agent.json: CaptionDraft with a 1-2 sentence caption and 4-6 hashtags.
- visual-director-agent.json: VisualBrief with a 2-3 sentence composition/mood/
  color brief.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- AI primitive matches the spec verbatim; an Agent is never silently swapped for
  AutonomousAgent or vice versa (Lesson 1).
- Workflow step timeouts set explicitly on every agent-calling step (Lesson 4).
- Optional<T> for every nullable lifecycle field on the view row record (Lesson 6).
- AutonomousAgent companion Tasks.java rule does not apply here (no AutonomousAgent),
  but if a future edit introduces one, add the Tasks file (Lesson 7).
- Verify model names against the provider's current lineup before locking
  (Lesson 8).
- Run command is "/akka:build" (Claude Code slash command), never "mvn akka:run"
  (Lesson 9).
- Explicit dev-mode http-port 9273 in application.conf (Lesson 10).
- source.platform and brand origin are corpus-internal — never user-facing
  (Lesson 11).
- UI fits the 1080px content column with no horizontal scroll (Lesson 12).
- Integration label is "Runs out of the box" — never T1/T2/T3/T4 or "deferred"
  (Lesson 13).
- emptyState() never calls commandContext() (Lesson 23 ref via Lesson 3).
- static-resources/index.html includes the mermaid CSS overrides and theme
  variables from Lesson 24 (state-label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- No forbidden words anywhere (Lesson 23 / authoring-guide Section 13).
- Tab switching matches by data-tab / data-panel attribute, never by NodeList
  index; no hidden zombie panels — delete removed tabs (Lesson 26).
- Single self-contained static-resources/index.html; no ui/ folder, no npm
  (Lesson 17).
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the mock option), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
