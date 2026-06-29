# SPEC — landing-page-builder

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Landing Page Generator.
**One-line pitch:** The user types a one-line product concept into the App UI; three agents run in sequence — analyze the concept, select a Tailwind template, customize it — and the UI shows a finished landing page with its generated markup.

## 2. What this blueprint demonstrates

The sequential-pipeline coordination pattern: each agent consumes the prior agent's typed output, with a durable workflow holding the in-flight state between stages. The governance pattern wires a before-tool-call guardrail (the template-selection agent may only scrape allowlisted, robots-permitted URLs), a before-agent-response guardrail (generated markup and copy are sanitized before they are stored or shown), and a build-gate that lints generated markup before a page can be published.

## 3. User-facing flows

1. The user opens the App UI tab and types a one-line concept (e.g. "a booking page for a mobile bike-repair service"). On Submit, the service starts a pipeline and returns a `pageId`. The page appears in `QUEUED`, then `ANALYZED`.
2. The pipeline advances on its own: `SELECTED` once a template is chosen, `CUSTOMIZED` once markup is generated, then `PUBLISHED` once the markup passes the lint gate. The UI updates live over SSE.
3. The user clicks a published page to see its brief, chosen template, and rendered markup preview.
4. If generated markup fails the lint gate, the page transitions to `REJECTED` with a reason; the UI shows it.
5. Without any interaction, `RequestSimulator` drips a fresh concept every 30 seconds, so pages keep arriving.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| IdeaAnalyst | AutonomousAgent | Concept → structured `ConceptBrief` | GenerationWorkflow | GenerationWorkflow |
| TemplateSelector | AutonomousAgent | Brief → `TemplateChoice` via search + scrape tools | GenerationWorkflow | ScraperEndpoint |
| CustomizationAgent | AutonomousAgent | Brief + template → `LandingPage` markup | GenerationWorkflow | GenerationWorkflow |
| GenerationWorkflow | Workflow | Orchestrates analyze → select → customize → validate | RequestConsumer | PageEntity, the three agents |
| PageEntity | EventSourcedEntity | Per-page lifecycle state | GenerationWorkflow | PagesView |
| InboundRequestQueue | EventSourcedEntity | Records each submitted concept | GenerationEndpoint, RequestSimulator | RequestConsumer |
| PagesView | View | List-of-pages read model | PageEntity | GenerationEndpoint |
| RequestConsumer | Consumer | Starts a workflow per queued concept | InboundRequestQueue | GenerationWorkflow |
| RequestSimulator | TimedAction | Drips a canned concept every 30s | sample-events | InboundRequestQueue |
| GenerationEndpoint | HttpEndpoint | `/api` surface — submit, list, single, SSE, metadata | UI | PageEntity, PagesView |
| ScraperEndpoint | HttpEndpoint | In-process template source returning canned markup for allowlisted URLs | TemplateSelector | sample-docs |
| AppEndpoint | HttpEndpoint | Serves `/` and `/app/*` | browser | static-resources |

Names are used verbatim by `/akka:specify`.

## 5. Data model

See `reference/data-model.md`. Entity state and View row is the `Page` record; every nullable lifecycle field is `Optional<T>` (Lesson 6). Status enum `PageStatus`: `QUEUED · ANALYZED · SELECTED · CUSTOMIZED · PUBLISHED · REJECTED`. Events: `ConceptAnalyzed`, `TemplateSelected`, `PageCustomized`, `PagePublished`, `PageRejected`. Agent result records: `ConceptBrief`, `TemplateChoice`, `LandingPage`.

## 6. API contract

See `reference/api-contract.md` for payload schemas and the SSE event form. Top-level surface:

```
POST /api/generate                 -> { pageId }
GET  /api/pages ?status=...        -> { pages: [Page, ...] }
GET  /api/pages/{pageId}           -> Page
GET  /api/pages/sse                -> Server-Sent Events of Page
GET  /api/scrape ?url=...          -> canned template markup (allowlisted URLs only)
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm). Browser title: `<title>Akka Sample: Landing Page Generator</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index (Lesson 26). Mermaid diagrams on the Architecture tab carry the state-label colour + edge-label `overflow:visible` CSS overrides (Lesson 24). See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1** — a before-tool-call guardrail on `TemplateSelector`'s scrape tool checks each target URL against an allowlist and a robots rule; off-list URLs are blocked and the run falls back to canned sources.
- **G2** — a before-agent-response guardrail on `CustomizationAgent` sanitizes generated markup and copy (strip script tags, inline event handlers, external form actions) before the result is stored.
- **A1** — the workflow's validate step runs an in-process markup lint; failing markup transitions the page to `REJECTED`. The Maven test phase is the deploy-time build gate.

## 9. Agent prompts

- `prompts/idea-analyst.md` — turns a one-line concept into a structured brief.
- `prompts/template-selector.md` — chooses a template by searching and scraping allowlisted sources.
- `prompts/customization-agent.md` — produces sanitized landing-page markup from the brief and template.

## 10. Acceptance

See `reference/user-journeys.md`. Passing journeys:

1. Submit a concept; within ~30s the page reaches `PUBLISHED` with non-empty markup.
2. A scrape to an off-allowlist URL is blocked; the run completes on canned sources.
3. Markup containing a `<script>` tag is sanitized before storage; the stored markup has no script.
4. Markup that fails the lint gate transitions to `REJECTED` with a reason.

---

## 11. Implementation directives

The whole SPEC.md — Sections 1–11 — is the input to `/akka:specify @SPEC.md`. This section carries the Akka-specific details Sections 1–10 don't repeat.

```
Create a sample named landing-page-builder demonstrating the
sequential-pipeline x sales-marketing cell. Runs out of the box (no external
services; search and scraping modeled in-process). Maven group io.akka.samples.
Maven artifact landing-page-builder. Java package
io.akka.samples.landingpagebuilder. Akka 3.6.0. HTTP port 9364.

Components to wire (exactly):
- 3 AutonomousAgents:
  - IdeaAnalyst: input a concept String, returns typed ConceptBrief{audience,
    valueProps(List<String>), tone, colorTheme, sections(List<String>)}.
  - TemplateSelector: input a ConceptBrief, returns typed TemplateChoice{
    templateName, templateUrl, rationale}. Declares a scrape capability whose
    tool calls GET /api/scrape?url=...; a before-tool-call guardrail checks the
    URL against an allowlist + robots rule before the tool runs.
  - CustomizationAgent: input ConceptBrief + TemplateChoice, returns typed
    LandingPage{title, html}. A before-agent-response guardrail sanitizes html
    (remove <script>, on* handlers, external form actions) before the result is
    returned.
  Each declares definition() with capability(TaskAcceptance.of(task)
  .maxIterationsPerTask(3)). Never silently downgrade to Agent (Lesson 1).
- 1 Workflow GenerationWorkflow with steps analyzeStep -> selectStep ->
  customizeStep -> validateStep. analyzeStep calls
  forAutonomousAgent(IdeaAnalyst.class,...).runSingleTask(...) then
  forTask(taskId).result(...); writes PageEntity.recordAnalysis. selectStep and
  customizeStep follow the same agent-call shape, writing recordTemplate and
  recordCustomization. validateStep runs an in-process HtmlValidator lint over
  the markup; on pass calls PageEntity.publish; on fail calls
  PageEntity.reject(reason). Override settings() with stepTimeout(60s) on
  analyzeStep, selectStep, customizeStep (Lesson 4); WorkflowSettings is nested
  in Workflow, no import (Lesson 5).
- 1 EventSourcedEntity PageEntity holding a Page record: id, concept
  (Optional<String>), PageStatus enum, and Optional lifecycle fields analyzedAt,
  brief, selectedAt, templateName, templateUrl, customizedAt, html, validatedAt,
  lintPassed, publishedAt, publishedUrl, rejectedAt, rejectReason. Events:
  ConceptAnalyzed, TemplateSelected, PageCustomized, PagePublished, PageRejected.
  Commands: recordAnalysis, recordTemplate, recordCustomization, publish, reject,
  getPage. emptyState() returns Page.initial("") with NO commandContext()
  reference (Lesson 3).
- 1 EventSourcedEntity InboundRequestQueue, single instance "default", one
  command enqueueConcept(concept) emitting ConceptQueued.
- 1 View PagesView with row type Page, table updater consuming PageEntity events.
  ONE query getAllPages: SELECT * AS pages FROM pages_view. No WHERE status
  filter (Akka cannot auto-index enum columns, Lesson 2) — filter client-side.
- 1 Consumer RequestConsumer subscribed to InboundRequestQueue events; on each
  ConceptQueued starts a GenerationWorkflow with a fresh UUID.
- 1 TimedAction RequestSimulator (every 30s) reads the next line from
  src/main/resources/sample-events/concepts.jsonl and calls
  InboundRequestQueue.enqueueConcept.
- 3 HttpEndpoints: GenerationEndpoint at /api (generate, pages list filtered
  client-side from getAllPages, single page, SSE stream, three /api/metadata/*
  endpoints serving files from src/main/resources/metadata/). ScraperEndpoint at
  /api/scrape returning canned markup from src/main/resources/sample-docs/ for
  allowlisted URLs and 403 otherwise. AppEndpoint serving / -> 302
  /app/index.html and /app/* -> static-resources/*.

Companion files:
- GenerationTasks.java declaring three Task<R> constants: ANALYZE
  (resultConformsTo ConceptBrief), SELECT (TemplateChoice), CUSTOMIZE
  (LandingPage). Required for every AutonomousAgent (Lesson 7).
- ConceptBrief, TemplateChoice, LandingPage records (see reference/data-model.md).
- HtmlValidator.java — pure helper: returns pass/fail + reason for generated
  markup (balanced tags, no script, has <title>, non-empty body).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9364 and agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/sample-events/concepts.jsonl with 8 canned concept lines.
- src/main/resources/sample-docs/ with 3-4 canned template markup files the
  ScraperEndpoint serves (allowlisted URLs map to these).
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md} —
  classpath copies for the metadata endpoints.
- eval-matrix.yaml + risk-survey.yaml at the project root.
- README.md at the project root (Lesson 20 — no governance-mechanisms section,
  no configuration section).
- src/main/resources/static-resources/index.html — single self-contained file,
  inline CSS + JS, runtime CDN imports for markdown + YAML allowed. Five tabs.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, detect which of ANTHROPIC_API_KEY / OPENAI_API_KEY /
  GOOGLE_AI_GEMINI_API_KEY is set. If exactly one, default application.conf's
  model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (IdeaAnalyst -> ConceptBrief, TemplateSelector -> TemplateChoice,
  CustomizationAgent -> LandingPage; see Mock LLM shapes below). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the reference.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option (a)), written to
src/main/resources/mock-responses/{idea-analyst,template-selector,
customization-agent}.json, 4-6 entries each:
- idea-analyst.json: ConceptBrief objects {audience, valueProps[], tone,
  colorTheme, sections[]}.
- template-selector.json: TemplateChoice objects {templateName, templateUrl
  (allowlisted), rationale}.
- customization-agent.json: LandingPage objects {title, html} where html is a
  short, already-sanitized Tailwind landing-page fragment.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: explicit stepTimeout on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable lifecycle field on the Page row record.
- Lesson 7: every AutonomousAgent has its GenerationTasks Task<R> constant.
- Lesson 8: verify model names are current before locking.
- Lesson 9: run command is "/akka:build" (Claude Code slash command), never
  "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9364 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box", never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: mermaid state-label colour + edge-label overflow:visible CSS
  overrides and theme variables in index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — a missing API key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
