# SPEC — hitl-story-crafting

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** HITL Story Crafting.
**One-line pitch:** A reader provides a story premise; `StoryDrafterAgent` writes the first chapter; the workflow pauses so the reader can provide direction for the next chapter; after each direction the agent writes the next chapter; repeating until the reader ends the story.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the content-editorial domain: a repeating 3-step graph that drafts a chapter, screens it for content policy, then waits for human direction before the next iteration. The governance pattern is an application-level human gate between every chapter and the next, and an output guardrail before each draft reaches the reader.

## 3. User-facing flows

1. A client POSTs a premise to `/api/stories`. The response returns `{ storyId }`. The story appears in the UI in `AWAITING_DIRECTION` once `StoryDrafterAgent` finishes the first chapter (typically 5–30 s), with the chapter title and body visible.
2. The reader provides a direction for the next chapter by POSTing to `/api/stories/{storyId}/continue`. The workflow writes the next chapter and returns to `AWAITING_DIRECTION`.
3. The reader ends the story by POSTing to `/api/stories/{storyId}/end`. The story transitions to terminal `COMPLETED` and no further chapters are written.
4. If `ContentGuardAgent` blocks a draft, the story transitions to terminal `SCREENED_OUT` and the reader is shown the reason.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| StoryDrafterAgent | AutonomousAgent | Writes the next chapter given the story so far and reader direction; returns `ChapterDraft{title, body}` | StoryWorkflow | StoryEntity |
| ContentGuardAgent | AutonomousAgent | Screens a chapter draft for content policy; returns `GuardResult{passed, reason}` | StoryWorkflow | StoryEntity |
| StoryWorkflow | Workflow | Orchestrates draft chapter → screen draft → await reader direction, repeating per turn | StoryEndpoint | StoryDrafterAgent, ContentGuardAgent, StoryEntity |
| StoryEntity | EventSourcedEntity | Holds the story state, chapter list, and lifecycle events | StoryWorkflow, StoryEndpoint | StoriesView |
| StoriesView | View | CQRS read model of all stories | StoryEntity | StoryEndpoint |
| StoryEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | StoryWorkflow, StoryEntity, StoriesView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Story` (StoryEntity state and StoriesView row): `id` (String), `premise` (`Optional<String>`), `status` (StoryStatus enum), `turnCount` (int), `chapters` (`List<Chapter>`), and lifecycle fields all `Optional<T>`: `startedAt`, `lastChapterAt`, `completedAt`, `screenedOutAt`, `screenOutReason`. Every nullable field is `Optional<T>` (Lesson 6).

`Chapter` (embedded record): `turnNumber` (int), `title` (String), `body` (String), `direction` (`Optional<String>`).

`StoryStatus` enum: `AWAITING_DIRECTION`, `DRAFTING`, `SCREENED_OUT`, `COMPLETED`.

Events: `StoryStarted`, `ChapterAdded`, `DirectionProvided`, `StoryCompleted`, `StoryScreenedOut`.

Domain records: `ChapterDraft(String title, String body)`, `ReaderDirection(String direction)`, `GuardResult(boolean passed, String reason)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/stories                          -> { storyId }
POST /api/stories/{storyId}/continue       -> 200 | 404 | 409
POST /api/stories/{storyId}/end            -> 200 | 404 | 409
GET  /api/stories                          -> { stories: [Story, ...] }
GET  /api/stories/{storyId}               -> Story | 404
GET  /api/stories/sse                      -> Server-Sent Events of Story
GET  /api/metadata/eval-matrix             -> text/yaml
GET  /api/metadata/risk-survey             -> text/yaml
GET  /api/metadata/readme                  -> text/markdown
GET  /                                     -> 302 /app/index.html
GET  /app/{*path}                          -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: HITL Story Crafting</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab starts a story with a premise, lists stories live via SSE, shows chapters in order, and presents a direction input plus Continue and End buttons on stories in `AWAITING_DIRECTION`. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `StoryWorkflow` pauses at the await-direction step; `/api/stories/{id}/continue` and `/api/stories/{id}/end` resume it.
- **G1 — guardrail · before-agent-response.** `ContentGuardAgent` screens each `ChapterDraft` before it is persisted for the reader. If the guard rejects the draft the workflow ends the story in `SCREENED_OUT`.

## 9. Agent prompts

- `StoryDrafterAgent` — writes the next chapter of an interactive story. See `prompts/story-drafter-agent.md`.
- `ContentGuardAgent` — screens a chapter draft for content policy compliance. See `prompts/content-guard-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Start a story.** POST a premise; within ~30 s a story appears in `AWAITING_DIRECTION` with a non-empty first chapter.
2. **Continue a story.** POST a direction to a story in `AWAITING_DIRECTION`; within ~30 s a new chapter appears and the story is back in `AWAITING_DIRECTION`.
3. **End a story.** POST end on a story in `AWAITING_DIRECTION`; it transitions to terminal `COMPLETED`.
4. **Guard blocks.** Submit a premise that triggers the content guard; the story transitions to `SCREENED_OUT` with a reason; no chapter is persisted.
5. **No chapter without direction.** While a story is in `AWAITING_DIRECTION` and the reader has not acted, the workflow waits; no new chapter is produced.

---

## 11. Implementation directives

```
Create a sample named hitl-story-crafting demonstrating the human-in-loop-gate ×
content-editorial cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-content-editorial-hitl-story-crafting.
Java package io.akka.samples.humanintheloopstorycrafting. Akka 3.6.0. HTTP port 9143.

Components to wire (exactly):
- 2 AutonomousAgents: StoryDrafterAgent (writes the next story chapter, returns a
  typed ChapterDraft{title,body}) and ContentGuardAgent (screens a draft for content
  policy, returns a typed GuardResult{passed,reason}). Each declares definition()
  returning an AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow StoryWorkflow with three tasks per turn: draftChapterStep ->
  screenDraftStep -> awaitDirectionStep. draftChapterStep calls
  forAutonomousAgent(StoryDrafterAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...), writes chapterAdded on StoryEntity.
  screenDraftStep calls ContentGuardAgent; if GuardResult.passed is false, calls
  storyScreenedOut on StoryEntity and ends the workflow. awaitDirectionStep reads
  StoryEntity.getStory; on AWAITING_DIRECTION it self-schedules a 5-second resume
  timer; on DRAFTING it loops back to draftChapterStep; on COMPLETED or SCREENED_OUT
  it ends. Override settings() with stepTimeout(60s) on draftChapterStep and
  screenDraftStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity StoryEntity holding a Story record with id, premise
  (Optional<String>), StoryStatus enum {AWAITING_DIRECTION,DRAFTING,SCREENED_OUT,
  COMPLETED}, turnCount (int), chapters (List<Chapter>), and Optional lifecycle
  fields (startedAt, lastChapterAt, completedAt, screenedOutAt, screenOutReason).
  Chapter embedded record: turnNumber (int), title (String), body (String), direction
  (Optional<String>). Events: StoryStarted, ChapterAdded, DirectionProvided,
  StoryCompleted, StoryScreenedOut. Commands: startStory, chapterAdded,
  directionProvided, storyCompleted, storyScreenedOut, getStory. emptyState()
  returns Story.initial("") with no commandContext() reference (Lesson 3).
- 1 View StoriesView with row type Story, table updater consuming StoryEntity events.
  ONE query: getAllStories SELECT * AS stories FROM stories_view. No WHERE status
  filter (Akka cannot auto-index enum columns, Lesson 2) — filter client-side in
  callers.
- 2 HttpEndpoints: StoryEndpoint at /api with start-story (POST /api/stories, starts
  a StoryWorkflow with a fresh UUID), continue (POST /api/stories/{id}/continue,
  calls directionProvided then transitions status to DRAFTING), end (POST
  /api/stories/{id}/end, calls storyCompleted), stories list (filter client-side from
  getAllStories), single story, SSE stream, and three /api/metadata/* endpoints
  serving the YAML/MD files from src/main/resources/metadata/. AppEndpoint serving
  / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- StoryTasks.java declaring two Task<R> constants: DRAFT_CHAPTER (resultConformsTo
  ChapterDraft) and SCREEN_DRAFT (resultConformsTo GuardResult).
- ChapterDraft(String title, String body), ReaderDirection(String direction),
  GuardResult(boolean passed, String reason).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9143
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (H1, G1) and a
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
  (StoryDrafterAgent -> ChapterDraft, ContentGuardAgent -> GuardResult; see
  src/main/resources/mock-responses/{story-drafter-agent,content-guard-agent}.json
  with 4–6 entries each). Sets model-provider = mock.
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
- story-drafter-agent.json: 4–6 entries, each { "title": "...", "body": "4–6
  paragraphs of plausible story prose" }.
- content-guard-agent.json: 4–6 entries, each { "passed": true, "reason": "..." }
  with at least one entry where passed is false and reason names the policy concern.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  StoryTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9143 declared in application.conf.
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
