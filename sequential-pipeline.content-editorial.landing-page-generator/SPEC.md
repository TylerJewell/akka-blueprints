# Landing Page Generator

A sequential agent pipeline that turns a one-line product concept into a complete landing page. The user types a concept (e.g. "a budgeting app for freelancers"); the system produces page copy, a section structure, and call-to-action blocks, then runs a brand-safety review before marking the page ready.

---

## 1. System name + one-line pitch

**Landing Page Generator.** Submit a product concept; three agents run in sequence to draft copy, organize it into page sections, and write call-to-action blocks; a brand-safety guardrail reviews the assembled page before it is marked ready for use.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern: a single `Workflow` drives three `AutonomousAgent`s in a fixed order, each consuming the previous stage's typed output, with no branching or fan-out. The governance pattern is a **guardrail at the before-agent-response hook** (control G1): the final public-facing copy is evaluated for accuracy and brand safety before the page can reach the `READY` state. A page that fails review transitions to `FLAGGED` instead, blocking it from being treated as finished.

## 3. User-facing flows

1. The user submits a concept through the App UI tab or `POST /api/concepts`. The response carries the `pageId`. A new `LandingPageEntity` appears in `DRAFTING_COPY`.
2. The pipeline runs. `CopyAgent` drafts headline, subhead, and body copy; the page moves to `STRUCTURING`. `StructureAgent` organizes the copy into ordered sections; the page moves to `WRITING_CTA`. `CtaAgent` writes call-to-action blocks; the page moves to `REVIEWING`.
3. The brand-safety guardrail evaluates the assembled page. If it passes, the page moves to `READY` with a review score. If it fails, the page moves to `FLAGGED` with a flag reason; the App UI shows the reason.
4. Without any user interaction, `ConceptSimulator` drips sample concepts from a canned file, so the page list fills on its own.

These flows are the acceptance journeys in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CopyAgent` | AutonomousAgent | Drafts headline/subhead/body copy from a concept | `LandingPageWorkflow` | `LandingPageEntity` |
| `StructureAgent` | AutonomousAgent | Organizes copy into ordered page sections | `LandingPageWorkflow` | `LandingPageEntity` |
| `CtaAgent` | AutonomousAgent | Writes call-to-action blocks | `LandingPageWorkflow` | `LandingPageEntity` |
| `LandingPageWorkflow` | Workflow | Orchestrates copy → structure → cta → review | `ConceptConsumer` | `CopyAgent`, `StructureAgent`, `CtaAgent`, `LandingPageEntity` |
| `LandingPageEntity` | EventSourcedEntity | Holds page state + lifecycle events | `LandingPageWorkflow` | `LandingPagesView` |
| `ConceptQueue` | EventSourcedEntity | Inbound concept queue | `LandingPageEndpoint`, `ConceptSimulator` | `ConceptConsumer` |
| `LandingPagesView` | View | CQRS read model of all pages | `LandingPageEntity` | `LandingPageEndpoint` |
| `ConceptConsumer` | Consumer | Starts a workflow per inbound concept | `ConceptQueue` | `LandingPageWorkflow` |
| `ConceptSimulator` | TimedAction | Drips canned concepts every 30s | (timer) | `ConceptQueue` |
| `LandingPageEndpoint` | HttpEndpoint | JSON API + SSE + metadata | UI / clients | `ConceptQueue`, `LandingPageEntity`, `LandingPagesView` |
| `AppEndpoint` | HttpEndpoint | Serves the static UI | browser | static-resources |

## 5. Data model

Full field list with nullability in `reference/data-model.md`. Authoritative shapes:

```java
public record LandingPage(
    String id,
    Optional<String> concept,
    PageStatus status,
    Optional<String> headline,
    Optional<String> subhead,
    Optional<String> bodyCopy,
    Optional<List<String>> sections,
    Optional<String> ctaPrimary,
    Optional<String> ctaSecondary,
    Optional<Integer> reviewScore,
    Optional<String> flagReason,
    Optional<String> copyDraftedAt,
    Optional<String> structuredAt,
    Optional<String> ctaWrittenAt,
    Optional<String> reviewedAt) {}

public enum PageStatus { DRAFTING_COPY, STRUCTURING, WRITING_CTA, REVIEWING, READY, FLAGGED }

// Agent result records
public record CopyDraft(String headline, String subhead, String bodyCopy) {}
public record PageSections(List<String> sections) {}
public record CtaBlocks(String primary, String secondary) {}
public record ReviewResult(boolean passed, int score, String reason) {}
```

Events: `CopyDrafted`, `PageStructured`, `CtaWritten`, `PageMarkedReady`, `PageFlagged`, `ConceptQueued`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

## 6. API contract

Top-level surface (full schemas in `reference/api-contract.md`):

```
POST /api/concepts                  -> { pageId }
GET  /api/pages                     -> { pages: [LandingPage, ...] }
GET  /api/pages/{pageId}            -> LandingPage
GET  /api/pages/sse                 -> Server-Sent Events of LandingPage
GET  /api/metadata/eval-matrix      -> text/yaml
GET  /api/metadata/risk-survey      -> text/yaml
GET  /api/metadata/readme           -> text/markdown
GET  /                              -> 302 /app/index.html
GET  /app/{*path}                   -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm). Browser title: `<title>Akka Sample: Landing Page Generator</title>`. Five tabs, switched **by `data-tab` / `data-panel` attribute, never NodeList index** (Lesson 26); no hidden zombie panels.

1. **Overview** — eyebrow "Overview" + headline + the four cards Try it / How it works / Components / API contract.
2. **Architecture** — the four mermaid diagrams from `PLAN.md`, with the state-label CSS overrides and theme variables from Lesson 24.
3. **Risk Survey** — `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style; `TO_BE_COMPLETED_BY_DEPLOYER` values muted.
4. **Eval Matrix** — `/api/metadata/eval-matrix` in the same style; the label column carries a colored mechanism pill (guardrail red).
5. **App UI** — concept submit box; live SSE list of pages with status and review score; a flag reason shown on `FLAGGED` pages; the rendered headline/sections/CTA on `READY` pages.

Details in `reference/ui-mockup.md`.

## 8. Governance

Controls in `eval-matrix.yaml`; deployer survey in `risk-survey.yaml`. One mechanism is wired:

- **G1 — brand-safety review (guardrail, before-agent-response, llm-content-evaluation, blocking).** Before `LandingPageWorkflow`'s `reviewStep` records the page as ready, the assembled copy is evaluated for accuracy and brand safety. A pass records `PageMarkedReady` with a score; a fail records `PageFlagged` with a reason and blocks the `READY` transition.

## 9. Agent prompts

- `prompts/copy-agent.md` — drafts headline, subhead, and body copy from the concept, and carries the brand-safety review rubric the guardrail applies.
- `prompts/structure-agent.md` — organizes copy into an ordered list of page sections.
- `prompts/cta-agent.md` — writes a primary and secondary call-to-action block.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`:

1. **Submit a concept and reach READY** — `POST /api/concepts`; the page runs copy → structure → cta → review and lands in `READY` with non-empty headline, sections, CTA, and a review score.
2. **A flagged page** — a concept that trips the brand-safety rubric lands in `FLAGGED` with a non-empty flag reason and is never marked `READY`.
3. **Background simulator** — with no UI interaction, `ConceptSimulator` seeds a concept within ~30s and a page appears.
4. **All five tabs render** — Overview, Architecture, Risk Survey, Eval Matrix, App UI all render; the Eval Matrix tab shows control G1 with a guardrail pill.

## 11. Implementation directives

```
Create a sample named landing-page-generator demonstrating the
sequential-pipeline × content-editorial cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact landing-page-generator.
Java package io.akka.samples.landingpagegenerator. Akka 3.6.0. HTTP port 9297.

Components to wire (exactly):
- 3 AutonomousAgents: CopyAgent (returns CopyDraft{headline,subhead,bodyCopy}),
  StructureAgent (returns PageSections{sections}), CtaAgent (returns
  CtaBlocks{primary,secondary}). Each declares definition() with
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Each extends
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow LandingPageWorkflow with steps copyStep -> structureStep ->
  ctaStep -> reviewStep. copyStep calls forAutonomousAgent(CopyAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...); records CopyDrafted on
  LandingPageEntity. structureStep and ctaStep follow the same form. reviewStep
  applies the brand-safety guardrail (control G1) over the assembled page: on
  pass, calls LandingPageEntity.markReady(score); on fail, calls
  markFlagged(reason) and ends. Override settings() with stepTimeout(60s) on
  copyStep, structureStep, ctaStep, and reviewStep; defaultStepRecovery
  maxRetries(2).failoverTo(error). WorkflowSettings is nested in Workflow — no
  import.
- 1 EventSourcedEntity LandingPageEntity holding a LandingPage record with id,
  concept (Optional<String>), PageStatus enum, and the Optional lifecycle
  fields (headline, subhead, bodyCopy, sections, ctaPrimary, ctaSecondary,
  reviewScore, flagReason, copyDraftedAt, structuredAt, ctaWrittenAt,
  reviewedAt). Events: CopyDrafted, PageStructured, CtaWritten, PageMarkedReady,
  PageFlagged. Commands: recordCopy, recordStructure, recordCta, markReady,
  markFlagged, getPage. emptyState() returns LandingPage.initial("") with no
  commandContext() reference.
- 1 EventSourcedEntity ConceptQueue with a single command enqueueConcept(concept)
  emitting ConceptQueued.
- 1 View LandingPagesView with row type LandingPage, table updater consuming
  LandingPageEntity events. ONE query: getAllPages SELECT * AS pages FROM
  pages_view. No WHERE status filter (Akka cannot auto-index enum columns) —
  filter client-side in callers.
- 1 Consumer ConceptConsumer subscribed to ConceptQueue events; on each event
  starts a LandingPageWorkflow with a fresh UUID.
- 1 TimedAction ConceptSimulator (every 30s, reads next line from
  src/main/resources/sample-events/concepts.jsonl and calls
  ConceptQueue.enqueueConcept).
- 2 HttpEndpoints: LandingPageEndpoint at /api with concepts (submit), pages
  list (filter client-side from getAllPages), single page, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- LandingPageTasks.java declaring three Task<R> constants: COPY
  (resultConformsTo CopyDraft), STRUCTURE (PageSections), CTA (CtaBlocks).
- CopyDraft(String headline, String subhead, String bodyCopy),
  PageSections(List<String> sections), CtaBlocks(String primary, String
  secondary), ReviewResult(boolean passed, int score, String reason).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9297 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Verify model names are current before locking.
- src/main/resources/sample-events/concepts.jsonl with 8 canned concept lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with control G1 and matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data classes, capability.*, model.*; marking jurisdictions, frameworks,
  organization fields, incidents, and data residency/retention as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored in this blueprint).
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports
  for markdown, YAML, and mermaid are acceptable. Five tabs: Overview,
  Architecture (4 mermaid diagrams), Risk Survey, Eval Matrix, App UI. Match the
  governance.html visual style (dark/yellow/Instrument Sans/dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (CopyAgent -> CopyDraft, StructureAgent -> PageSections, CtaAgent ->
  CtaBlocks; see src/main/resources/mock-responses/{copy-agent,structure-agent,
  cta-agent}.json with 4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent response shapes when option (a) is chosen:
- CopyAgent -> CopyDraft{headline:string, subhead:string, bodyCopy:string}.
- StructureAgent -> PageSections{sections:[string,...] of 4-6 entries}.
- CtaAgent -> CtaBlocks{primary:string, secondary:string}.
- The review step's mock returns ReviewResult{passed:true, score:int 70-95,
  reason:""} for most concepts, and a passed:false/score<60/non-empty-reason
  result for at least one canned concept so the FLAGGED path is exercised.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: workflow step timeouts set explicitly (60s) on every agent-calling
  step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion LandingPageTasks.java.
- Lesson 8: verify model names are current before writing application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9297 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box", never tier
  codes or "deferred".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-diagram label CSS overrides + theme variables
  (transitionLabelColor #cccccc, edge-label foreignObject overflow:visible).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never by
  NodeList index; delete removed panels, do not display:none them.
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9297/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step runs longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without the user — a missing/unresolvable API key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
