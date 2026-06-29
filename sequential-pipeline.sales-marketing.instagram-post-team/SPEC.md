# SPEC — instagram-post-team

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 are the input; Section 12 chains the rest of the workflow.

---

## 1. System name + pitch

**System name:** Instagram Post Team.
**One-line pitch:** A marketer types one creative brief into the App UI; a pipeline of agents writes an Instagram caption with hashtags and an image-generation prompt, runs a brand and platform-policy check on the content, and marks the post ready when it passes.

## 2. What this blueprint demonstrates

The coordination pattern is a sequential pipeline: a workflow drives a fixed order of agent steps, each consuming the prior step's typed output. The governance pattern wires one before-agent-response guardrail — before either agent's result is persisted, a brand-voice and platform-policy check runs; content that breaks brand voice or platform policy is blocked from being marked ready, and the post lands in a blocked state with the reason recorded.

## 3. User-facing flows

1. The marketer submits a brief (one line of creative direction). The system creates a post in `COMPOSING` and returns its id.
2. The caption agent writes a caption with hashtags; the before-agent-response guardrail checks it. On pass the post moves to `PROMPTING`; on fail it moves to `BLOCKED`.
3. The image-prompt agent writes an image-generation prompt; the same guardrail checks it. On pass the post moves to `READY`; on fail it moves to `BLOCKED`.
4. The marketer reads the finished post content — image prompt, caption, hashtags — in the App UI. A blocked post shows the block reason instead.

These map to `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| CaptionAgent | Agent | Write an Instagram caption + hashtags from the brief | PostWorkflow | PostWorkflow |
| ImagePromptAgent | Agent | Write an image-generation prompt from brief + caption | PostWorkflow | PostWorkflow |
| PostWorkflow | Workflow | Orchestrate caption → image prompt → brand/safety check → ready | BriefConsumer | PostEntity |
| PostEntity | EventSourcedEntity | Hold the post content and its lifecycle | PostWorkflow, PostEndpoint | PostsView |
| BriefQueue | EventSourcedEntity | Inbound queue of submitted briefs | PostEndpoint, BriefSimulator | BriefConsumer |
| PostsView | View | CQRS read model of all posts | PostEntity | PostEndpoint |
| BriefConsumer | Consumer | Start one PostWorkflow per queued brief | BriefQueue | PostWorkflow |
| BriefSimulator | TimedAction | Drip canned briefs from a JSONL file | — | BriefQueue |
| PostEndpoint | HttpEndpoint | REST + SSE + metadata | UI | BriefQueue, PostEntity, PostsView |
| AppEndpoint | HttpEndpoint | Serve the static UI | browser | — |

## 5. Data model

Authoritative — `/akka:implement` writes records exactly as specified. See `reference/data-model.md`.

`PostContent` record fields: `id` String, `brief` String, `status` PostStatus enum, and these lifecycle fields, each `Optional<T>` (Lesson 6) except `hashtags`: `composedAt` Instant, `caption` String, `hashtags` List<String> (empty list when absent, never null), `promptedAt` Instant, `imagePrompt` String, `safetyCheckPassed` Boolean, `safetyCheckNotes` String, `readyAt` Instant, `blockedAt` Instant, `blockReason` String.

`PostStatus` enum: `COMPOSING`, `PROMPTING`, `READY`, `BLOCKED`, `FAILED`.

Events: `BriefQueued`, `CaptionWritten`, `ImagePromptWritten`, `PostReady`, `PostBlocked`.

Typed agent results: `CaptionDraft(String caption, List<String> hashtags)`, `ImagePrompt(String prompt)`.

## 6. API contract

Inline surface (full schemas in `reference/api-contract.md`):

```
POST /api/briefs               -> { postId }
GET  /api/posts                -> { posts: [PostContent, ...] }
GET  /api/posts/{postId}       -> PostContent
GET  /api/posts/sse            -> Server-Sent Events of PostContent
GET  /api/metadata/eval-matrix -> text/yaml
GET  /api/metadata/risk-survey -> text/yaml
GET  /api/metadata/readme      -> text/markdown
GET  /                         -> 302 /app/index.html
GET  /app/{*path}              -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Instagram Post Team</title>`. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, and removed tabs are deleted from the DOM rather than hidden (Lesson 26). Architecture tab renders the mermaid diagrams with the theme variables and CSS label overrides from Lesson 24. App UI tab: submit a brief, live SSE list of posts, each card showing caption, hashtags, image prompt, status pill, and block reason when blocked. See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1 — before-agent-response guardrail:** before the CaptionAgent and ImagePromptAgent results are persisted, a brand-voice and platform-policy check runs; failing content is blocked and the post transitions to `BLOCKED` with the reason recorded.
- **A1 — ci-gate:** tests must pass before deploy.

## 9. Agent prompts

- `prompts/caption-agent.md` — write an on-brand Instagram caption with hashtags from the brief.
- `prompts/image-prompt-agent.md` — write an image-generation prompt an image model can execute, consistent with the brief and caption.

## 10. Acceptance

Inline journeys (full set in `reference/user-journeys.md`):

1. **Submit a brief and watch it compose.** POST `/api/briefs`; the post appears via SSE, moves through `COMPOSING` and `PROMPTING`, and lands in `READY` with a non-empty caption, non-empty hashtags, and a non-empty image prompt.
2. **Blocked content.** A brief that drives off-brand or off-policy output transitions the post to `BLOCKED` with a non-empty `blockReason`, and the content is not marked ready.
3. **Background load from the simulator.** Without any UI interaction, `BriefSimulator` seeds new briefs from the JSONL file and the pipeline runs each to completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named instagram-post-team demonstrating the
sequential-pipeline x sales-marketing cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact instagram-post-team.
Java package io.akka.samples.instagrampostteam. Akka 3.6.0. HTTP port 9450.

Components to wire (exactly):
- 2 Agents (request/response, extends akka.javasdk.agent.Agent):
  - CaptionAgent: method write(String brief) returning Effect<CaptionDraft>.
    Uses effects().systemMessage(prompt).userMessage(brief)
    .responseAs(CaptionDraft.class).thenReply().
  - ImagePromptAgent: method draw(ImagePromptInput) returning Effect<ImagePrompt>.
  Each loads its system prompt from prompts/<agent>.md as a classpath resource.
- 1 Workflow PostWorkflow with steps: captionStep -> imagePromptStep ->
  finalizeStep. captionStep calls
  forAgent(CaptionAgent.class).inSession(postId).method(CaptionAgent::write)
  .invoke(brief), applies the before-agent-response guardrail to the result; on
  pass writes CaptionWritten and transitions to imagePromptStep; on fail writes
  PostBlocked and ends. imagePromptStep calls ImagePromptAgent, applies the same
  guardrail; on pass writes ImagePromptWritten and transitions to finalizeStep;
  on fail writes PostBlocked and ends. finalizeStep writes PostReady. Override
  settings() with stepTimeout(60s) on captionStep and imagePromptStep and
  defaultStepRecovery(maxRetries(2).failoverTo(PostWorkflow::error)).
- 1 EventSourcedEntity PostEntity holding a PostContent record with id, brief,
  PostStatus enum, and the Optional lifecycle fields listed in Section 5
  (hashtags is List<String> defaulting to empty, never null). Events:
  CaptionWritten, ImagePromptWritten, PostReady, PostBlocked. Commands:
  recordCaption, recordImagePrompt, markReady, markBlocked, getPost.
  emptyState() returns PostContent.initial("", "") with placeholder identity and
  no commandContext() reference.
- 1 EventSourcedEntity BriefQueue with a single command enqueue(brief) emitting
  BriefQueued.
- 1 View PostsView with row type PostContent, table updater consuming PostEntity
  events. ONE query: getAllPosts SELECT * AS posts FROM posts_view. No WHERE
  status filter (Akka cannot auto-index enum columns) — filter client-side in
  callers.
- 1 Consumer BriefConsumer subscribed to BriefQueue events; on each event starts
  a PostWorkflow with a fresh UUID.
- 1 TimedAction BriefSimulator (every 30s, reads next line from
  src/main/resources/sample-events/briefs.jsonl and calls BriefQueue.enqueue).
- 1 HttpEndpoint PostEndpoint at /api: briefs, posts list (filter client-side),
  single post, SSE stream, and three /api/metadata/* endpoints serving the
  YAML/MD files from src/main/resources/metadata/.
- 1 HttpEndpoint AppEndpoint: / -> 302 /app/index.html and /app/* ->
  static-resources/*.

Companion files:
- Records: CaptionDraft(String caption, List<String> hashtags),
  ImagePrompt(String prompt), ImagePromptInput(String brief, String caption).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9450
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/briefs.jsonl with 8 canned brief lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1 and A1 and a matching
  simplified_view list. No regulation_anchors.
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
  (CaptionAgent -> CaptionDraft, ImagePromptAgent -> ImagePrompt; write
  src/main/resources/mock-responses/{caption-agent,image-prompt-agent}.json with
  4-6 entries each). Sets model-provider = mock.
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
- caption-agent.json: CaptionDraft with a 1-2 sentence caption and 4-6 hashtags.
- image-prompt-agent.json: ImagePrompt with a 2-3 sentence image-generation
  prompt describing subject, composition, mood, and color.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- AI primitive matches the spec verbatim; an Agent is never silently swapped for
  AutonomousAgent or vice versa (Lesson 1).
- View queries never filter on an enum column; filter client-side (Lesson 2).
- emptyState() never calls commandContext() (Lesson 3).
- Workflow step timeouts set explicitly on every agent-calling step (Lesson 4).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- Optional<T> for every nullable lifecycle field on the view row record (Lesson 6).
- AutonomousAgent companion Tasks.java rule does not apply here (no AutonomousAgent),
  but if a future edit introduces one, add the Tasks file (Lesson 7).
- Verify model names against the provider's current lineup before locking
  (Lesson 8).
- Run command is "/akka:build" (Claude Code slash command), never "mvn akka:run"
  (Lesson 9).
- Explicit dev-mode http-port 9450 in application.conf (Lesson 10).
- source.platform and brand origin are corpus-internal — never user-facing
  (Lesson 11).
- UI fits the 1080px content column with no horizontal scroll (Lesson 12).
- Integration label is "Runs out of the box" — never T1/T2/T3/T4 or "deferred"
  (Lesson 13).
- static-resources/index.html includes the mermaid CSS overrides and theme
  variables from Lesson 24 (state-label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Single self-contained static-resources/index.html; no ui/ folder, no npm
  (Lesson 17).
- No forbidden words anywhere (Lesson 23 / authoring-guide Section 13).
- Key-sourcing follows Lesson 25; the key value is never written to disk.
- Tab switching matches by data-tab / data-panel attribute, never by NodeList
  index; no hidden zombie panels — delete removed tabs (Lesson 26).
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
