# SPEC — publishing

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** HITL Blog Publishing.
**One-line pitch:** A user submits a topic; `ContentAgent` drafts a blog post; the workflow pauses at an approval task; a human approves or rejects through the API; on approval `PublishingAgent` publishes and returns a URL.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the content-editorial domain: a 3-task graph that drafts, then waits at an unassigned approval task that a person completes through the API, then publishes only if approved. The governance pattern is an application-level human approval gate between the draft and publish phases, an output guardrail before a draft reaches the reviewer, and a tool-permission guardrail that blocks the publish step unless the post is approved.

## 3. User-facing flows

1. A client POSTs a topic to `/api/publish-request`. The response returns `{ postId }`. The post appears in the UI in `DRAFTED` once `ContentAgent` finishes (typically 5–30 s), with the drafted title and content visible.
2. The reviewer reads the draft and clicks Approve. This POSTs to `/api/posts/{postId}/approve`. The workflow resumes, `PublishingAgent` runs, and the published URL appears with status `PUBLISHED`.
3. The reviewer clicks Reject with a reason. This POSTs to `/api/posts/{postId}/reject`. The post moves to terminal `REJECTED` and the reason is shown. The publish step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ContentAgent | AutonomousAgent | Drafts a blog post; returns `DraftPost{title, content}` | PublishingWorkflow | PostEntity |
| PublishingAgent | AutonomousAgent | Publishes an approved post; returns `PublishedPost{url, publishedAt}` | PublishingWorkflow | PostEntity |
| PublishingWorkflow | Workflow | Orchestrates draft → await approval → publish | PublishingEndpoint | ContentAgent, PublishingAgent, PostEntity |
| PostEntity | EventSourcedEntity | Holds the post state and lifecycle events | PublishingWorkflow, PublishingEndpoint | PostsView |
| PostsView | View | CQRS read model of all posts | PostEntity | PublishingEndpoint |
| PublishingEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | PublishingWorkflow, PostEntity, PostsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Post` (PostEntity state and PostsView row): `id` (String), `topic` (`Optional<String>`), `status` (PostStatus enum), and lifecycle fields all `Optional<T>`: `draftedAt`, `draftTitle`, `draftContent`, `approvedAt`, `approvedBy`, `approverComment`, `rejectedAt`, `rejectedBy`, `rejectReason`, `publishedAt`, `publishedUrl`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`PostStatus` enum: `DRAFTED`, `APPROVED`, `REJECTED`, `PUBLISHED`.

Events: `PostDrafted`, `PostApproved`, `PostRejected`, `PostPublished`.

Domain records: `DraftPost(String title, String content)`, `ApprovalDecision(String approvedBy, String comment)`, `PublishedPost(String url, String publishedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/publish-request          -> { postId }
POST /api/posts/{postId}/approve   -> 200 | 404
POST /api/posts/{postId}/reject    -> 200 | 404
GET  /api/posts                    -> { posts: [Post, ...] }
GET  /api/posts/{postId}           -> Post
GET  /api/posts/sse                -> Server-Sent Events of Post
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: HITL Blog Publishing</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a topic, lists posts live via SSE, and shows Approve/Reject buttons on `DRAFTED` posts that have content. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `PublishingWorkflow` pauses at the await-approval task; `/api/posts/{id}/approve` and `/api/posts/{id}/reject` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `PublishingAgent` verifies `PostEntity.status == APPROVED` before the simulated publish tool runs.
- **G2 — guardrail · before-agent-response.** A guardrail on `ContentAgent` checks length and topic adherence before the draft is persisted for review.

## 9. Agent prompts

- `ContentAgent` — drafts a blog post on a topic. See `prompts/content-agent.md`.
- `PublishingAgent` — publishes an approved post and returns its URL. See `prompts/publishing-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Draft a post.** POST a topic; within ~30 s a post appears in `DRAFTED` with non-empty `draftContent`.
2. **Approve and publish.** Approve a `DRAFTED` post; it reaches `PUBLISHED` with a non-null `publishedUrl` within ~30 s.
3. **Reject a draft.** Reject a `DRAFTED` post with a reason; it moves to terminal `REJECTED` and the reason shows.
4. **Publish guard.** The publish step is never reached for a post that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named publishing demonstrating the human-in-loop-gate ×
content-editorial cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact publishing. Java package io.akka.samples.publishing.
Akka 3.6.0. HTTP port 9822.

Components to wire (exactly):
- 2 AutonomousAgents: ContentAgent (drafts a blog post, returns a typed
  DraftPost{title,content}) and PublishingAgent (returns a typed
  PublishedPost{url,publishedAt}). Each declares definition() returning an
  AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow PublishingWorkflow with three tasks: draftStep -> awaitApprovalStep
  -> publishStep. draftStep calls forAutonomousAgent(ContentAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordDraft on
  PostEntity. awaitApprovalStep polls PostEntity.getPost; on DRAFTED it
  self-schedules a 5-second resume timer; on APPROVED it transitions to
  publishStep; on REJECTED it ends. publishStep calls PublishingAgent and writes
  recordPublication. Override settings() with stepTimeout(60s) on draftStep and
  publishStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity PostEntity holding a Post record with id, topic
  (Optional<String>), PostStatus enum {DRAFTED,APPROVED,REJECTED,PUBLISHED}, and
  Optional lifecycle fields (draftedAt, draftTitle, draftContent, approvedAt,
  approvedBy, approverComment, rejectedAt, rejectedBy, rejectReason, publishedAt,
  publishedUrl). Events: PostDrafted, PostApproved, PostRejected, PostPublished.
  Commands: recordDraft, approve, reject, recordPublication, getPost. emptyState()
  returns Post.initial("") with no commandContext() reference (Lesson 3).
- 1 View PostsView with row type Post, table updater consuming PostEntity events.
  ONE query: getAllPosts SELECT * AS posts FROM posts_view. No WHERE status filter
  (Akka cannot auto-index enum columns, Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: PublishingEndpoint at /api with publish-request (starts a
  PublishingWorkflow with a fresh UUID), approve, reject, posts list (filter
  client-side from getAllPosts), single post, SSE stream, and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/. AppEndpoint
  serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- PublishingTasks.java declaring two Task<R> constants: DRAFT (resultConformsTo
  DraftPost) and PUBLISH (resultConformsTo PublishedPost).
- DraftPost(String title, String content), ApprovalDecision(String approvedBy,
  String comment), PublishedPost(String url, String publishedAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9822
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1, G2, H1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix cell,
  integration descriptor, how to run, the 5 tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ContentAgent -> DraftPost, PublishingAgent -> PublishedPost; see
  src/main/resources/mock-responses/{content-agent,publishing-agent}.json with 4–6
  entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option a):
- content-agent.json: 4–6 entries, each { "title": "...", "content": "4–6
  paragraphs of plausible blog prose" }.
- publishing-agent.json: 4–6 entries, each { "url":
  "https://blog.example.com/<slug>", "publishedAt": "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  PublishingTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9822 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid CSS overrides and theme variables
  (state-label colour, edge-label overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching matches by data-tab/data-panel attribute, never
  NodeList index; no hidden zombie panels in the DOM.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
