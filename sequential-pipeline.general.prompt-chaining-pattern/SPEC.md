# SPEC — prompt-chaining-workflow

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Prompt Chaining Workflow.
**One-line pitch:** A user submits a writing prompt; one `DraftingAgent` walks it through three chained steps — **OUTLINE** a document structure, **DRAFT** body content per section, **REFINE** the draft for clarity and citation — with a response guardrail validating the composite result before it reaches the caller and an on-decision evaluator scoring each refined document for structural completeness.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a general-purpose document-drafting domain. One `DraftingAgent` (AutonomousAgent) carries every LLM call across the chain. The pattern's defining property is the **typed step handoff**: the OUTLINE step's typed output (`Outline`) becomes the DRAFT step's instruction context; the DRAFT step's output (`Draft`) becomes the REFINE step's instruction context. The agent never holds all three phases in one conversation — each step runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** sits at the output boundary. After `refineStep` returns a `RefinedDocument`, the guardrail runs before the result is committed to `DraftEntity` or surfaced to the caller. It checks that the document meets minimum structural requirements: section count ≥ 2, total word count ≥ 150, and at least one citation present. If any check fails, the guardrail returns a structured rejection to the workflow, which retries `refineStep` within a 2-retry budget. The guardrail's rejection payload names the failed check precisely so the agent can self-correct.
- An **`on-decision-eval`** runs immediately after `RefinementApplied` lands, as `evalStep` inside the workflow. A deterministic, rule-based `QualityScorer` (no LLM call — the eval is rule-based on purpose, so the same document always scores the same) checks that every `Outline.section` has a matching `Draft.sectionDraft`, that every `sectionDraft` was incorporated into the `RefinedDocument`, that every cited reference in the refined document appears in the draft's `citations` list, and that the section count is consistent across all three typed outputs.

The blueprint shows that a sequential LLM chain is not just a series of prompts — the step-boundary handoffs enforce the dependency contract and the output guardrail is the right enforcement point for composite-result quality.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **prompt** into the input (or picks one of three seeded prompts — `Introduction to vector databases`, `Overview of transformer attention mechanisms`, `Guide to event-driven architecture patterns`).
2. The user clicks **Start drafting**. The UI POSTs to `/api/drafts` and receives a `draftId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `OUTLINING` — the workflow has started `outlineStep` and the agent has been handed the OUTLINE task.
4. Within ~10–20 s the card reaches `DRAFTING` — the typed `Outline` is visible in the card detail (a section list with headings and brief descriptions). The agent's OUTLINE task returned; the workflow recorded `OutlineProduced` and ran the DRAFT task.
5. Within ~15–30 s more the card reaches `REFINING`. The `Draft` is visible (per-section body paragraphs with a citations list).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `RefinedDocument` — title, abstract, per-section `RefinedSection` with polished body text and inline citation markers — plus a quality score chip (1–5) and a one-line rationale.
7. The user can submit another prompt; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DraftEndpoint` | `HttpEndpoint` | `/api/drafts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `DraftEntity`, `DraftView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DraftEntity` | `EventSourcedEntity` | Per-draft lifecycle: created → outlining → outlined → drafting → drafted → refining → refined → evaluated. Source of truth. | `DraftEndpoint`, `DraftingWorkflow` | `DraftView` |
| `DraftingWorkflow` | `Workflow` | One workflow per draft. Steps: `outlineStep` → `draftStep` → `refineStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `DraftEndpoint` after `CREATED` | `DraftingAgent`, `DraftEntity` |
| `DraftingAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `DraftingTasks.java`: `OUTLINE_DOCUMENT` → `Outline`, `DRAFT_DOCUMENT` → `Draft`, `REFINE_DOCUMENT` → `RefinedDocument`. Each task is registered with the step-appropriate function tools. | invoked by `DraftingWorkflow` | returns typed results |
| `OutlineTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `structurePrompt(prompt)` and `fetchSectionTemplates(domain)`. Reads from `src/main/resources/sample-data/templates/*.json`. | called from OUTLINE task | returns `List<Section>` |
| `DraftTools` | function-tools class | Implements `writeSection(section, context)` and `gatherCitations(topic)`. Pure in-memory transformations drawing from `src/main/resources/sample-data/citations/*.json`. | called from DRAFT task | returns `SectionDraft` / `List<Citation>` |
| `RefineTools` | function-tools class | Implements `polishSection(sectionDraft)` and `formatCitation(citation)`. | called from REFINE task | returns `RefinedSection` / `String` |
| `OutputGuardrail` | `before-agent-response` guardrail (registered on `DraftingAgent`) | Validates the `RefinedDocument` returned by `refineStep` before it is committed. Checks section count ≥ 2, word count ≥ 150, and ≥ 1 citation. On reject, returns a structured error to the workflow which retries `refineStep`. | every agent task response from REFINE step | accept / structured-reject |
| `QualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `RefinedDocument`, `Draft`, `Outline`. Output: `QualityResult{score, rationale}`. | called from `evalStep` | returns score |
| `DraftView` | `View` | Read model: one row per draft for the UI. | `DraftEntity` events | `DraftEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Section(String sectionId, String heading, String description) {}

record Outline(List<Section> sections, String documentTitle, Instant outlinedAt) {}

record Citation(String citationId, String label, String url, String snippet) {}

record SectionDraft(String sectionId, String heading, String body, List<String> citationIds) {}

record Draft(
    List<SectionDraft> sectionDrafts,
    List<Citation> citations,
    Instant draftedAt
) {}

record RefinedSection(String sectionId, String heading, String body, List<String> citationMarkers) {}

record RefinedDocument(
    String title,
    String abstract_,
    List<RefinedSection> sections,
    List<Citation> bibliography,
    Instant refinedAt
) {}

record QualityResult(
    int score,           // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record GuardrailRejection(
    String check,
    String reason,
    Instant rejectedAt
) {}

record DraftRecord(
    String draftId,
    Optional<String> prompt,
    Optional<Outline> outline,
    Optional<Draft> draft,
    Optional<RefinedDocument> refinedDocument,
    Optional<QualityResult> quality,
    DraftStatus status,
    List<GuardrailRejection> guardrailRejections,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DraftStatus {
    CREATED, OUTLINING, OUTLINED, DRAFTING, DRAFTED,
    REFINING, REFINED, EVALUATED, FAILED
}
```

Events on `DraftEntity`: `DraftCreated`, `OutlineStarted`, `OutlineProduced`, `DraftStarted`, `DraftWritten`, `RefineStarted`, `RefinementApplied`, `QualityScored`, `GuardrailRejected`, `DraftFailed`.

Every nullable lifecycle field on the `DraftRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/drafts` — body `{ prompt }` → `{ draftId }`.
- `GET /api/drafts` — list all drafts, newest-first.
- `GET /api/drafts/{id}` — one draft.
- `GET /api/drafts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Prompt Chaining Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of drafts (status pill + prompt + age) and a right pane with the selected draft's detail — prompt, outline sections list, draft body panels, refined document text, quality score chip, and a guardrail-rejection log strip if any response-guardrail rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (output-quality gate)**: `OutputGuardrail` is registered on `DraftingAgent` and runs after every task response from the REFINE step before the result is committed to `DraftEntity`. The accept rule is precise: `RefinedDocument.sections.size() ≥ 2` (section count), sum of `section.body.split(" ").length` across all sections ≥ 150 (word count), and `bibliography.size() ≥ 1` (citation presence). On reject, the guardrail returns a structured `quality-violation` error naming the failed check to the workflow. The workflow retries `refineStep` within a 2-retry budget. On each retry, the rejection reason is appended to the refine task's instruction context so the agent can address the specific gap. The workflow records a `GuardrailRejected{check, reason}` event for visibility. If the retry budget is exhausted, the workflow transitions to `FAILED`.
- **E1 — `on-decision-eval`**: runs immediately after `RefinementApplied` lands, as `evalStep` inside the workflow. `QualityScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every `Outline.section` must have a matching `SectionDraft` in the `Draft` (outline coverage), every `SectionDraft` must be incorporated into a `RefinedSection` in the `RefinedDocument` (draft coverage), every `Citation` cited in the `RefinedDocument.bibliography` must have appeared in `Draft.citations` (no hallucinated references), and the section count must be consistent across all three outputs (structural parity). Emits `QualityScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `DraftingAgent` → `prompts/drafting-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that step's tool set, treat each task's typed input as the entire context for that step, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded prompt `Introduction to vector databases`; within 90 s the draft reaches `EVALUATED` with ≥ 2 outline sections, ≥ 2 draft sections, ≥ 2 refined sections, and a quality score chip on the card.
2. **J2** — The agent's first REFINE iteration returns a `RefinedDocument` with only 1 section (mock LLM path). `OutputGuardrail` rejects it; a `GuardrailRejected` event lands on the entity; the workflow retries `refineStep`; the draft eventually completes correctly. The UI's rejection-log strip shows the one rejected response.
3. **J3** — A draft whose mock-LLM trajectory produces a bibliography entry absent from `Draft.citations` is scored 1 with a rationale naming the hallucinated reference; the UI flags the card.
4. **J4** — Each task's instructions and tool calls are visible in the per-draft trace (logged at `INFO`); the OUTLINE task's log shows only OUTLINE-tool calls, the DRAFT task shows only DRAFT-tool calls, the REFINE task shows only REFINE-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named prompt-chaining-workflow demonstrating the sequential-pipeline x general
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-general-prompt-chaining-pattern. Java package
io.akka.samples.promptchainingworkflow. Akka 3.6.0. HTTP port 9999.

Components to wire (exactly):

- 1 AutonomousAgent DraftingAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/drafting-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  OUTLINE, DRAFT, and REFINE tool sets are ALL registered on the agent; tool-step scoping is
  the job of OutputGuardrail for the composite result, NOT of conditional .tools(...) wiring.
  The before-agent-response guardrail (OutputGuardrail) is registered on the agent via the
  agent's guardrail-configuration block. On guardrail rejection the workflow retries refineStep
  within a 2-retry budget.

- 1 Workflow DraftingWorkflow per draftId with four steps:
  * outlineStep — emits OutlineStarted on the entity, then calls componentClient
    .forAutonomousAgent(DraftingAgent.class, "agent-" + draftId).runSingleTask(
      TaskDef.instructions("Prompt: " + prompt + "\nStep: OUTLINE\nStructure the prompt
      into 2-5 logical sections.")
        .metadata("draftId", draftId)
        .metadata("step", "OUTLINE")
        .taskType(DraftingTasks.OUTLINE_DOCUMENT)
    ). Reads forTask(taskId).result(OUTLINE_DOCUMENT) to get Outline. Writes
    DraftEntity.recordOutline(outline). WorkflowSettings.stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(outline, prompt)) and metadata.step = "DRAFT", taskType
    DRAFT_DOCUMENT. Writes DraftEntity.recordDraft(draft). stepTimeout 90s.
  * refineStep — emits RefineStarted, then runSingleTask with TaskDef.instructions
    (formatRefineContext(draft, outline, prompt)) and metadata.step = "REFINE", taskType
    REFINE_DOCUMENT. The OutputGuardrail runs on the agent response before it lands. On
    guardrail reject, appends the rejection reason to the next retry's instruction context.
    Writes DraftEntity.recordRefinement(refinedDocument). stepTimeout 90s.
    maxRetries(2).failoverTo(DraftingWorkflow::error).
  * evalStep — runs the deterministic QualityScorer over (refinedDocument, draft, outline)
    and writes DraftEntity.recordQuality(quality). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(DraftingWorkflow::error). The error step writes DraftFailed and
  ends.

- 1 EventSourcedEntity DraftEntity (one per draftId). State DraftRecord{draftId,
  prompt: Optional<String>, outline: Optional<Outline>, draft: Optional<Draft>,
  refinedDocument: Optional<RefinedDocument>, quality: Optional<QualityResult>,
  status: DraftStatus, guardrailRejections: List<GuardrailRejection>, createdAt: Instant,
  finishedAt: Optional<Instant>}. DraftStatus enum: CREATED, OUTLINING, OUTLINED, DRAFTING,
  DRAFTED, REFINING, REFINED, EVALUATED, FAILED. Events: DraftCreated{prompt}, OutlineStarted,
  OutlineProduced{outline}, DraftStarted, DraftWritten{draft}, RefineStarted,
  RefinementApplied{refinedDocument}, QualityScored{quality}, GuardrailRejected{check, reason},
  DraftFailed{reason}. Commands: create, startOutline, recordOutline, startDraft, recordDraft,
  startRefine, recordRefinement, recordQuality, recordGuardrailRejection, fail, getDraft.
  emptyState() returns DraftRecord.initial("") with all Optional fields as Optional.empty()
  and guardrailRejections = List.of() and no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside the
  event-applier.

- 1 View DraftView with row type DraftRow that mirrors DraftRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes DraftEntity events. ONE query
  getAllDrafts: SELECT * AS drafts FROM draft_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * DraftEndpoint at /api with POST /drafts (body {prompt}; mints draftId; calls
    DraftEntity.create(prompt); then starts DraftingWorkflow with id "drafting-" + draftId;
    returns {draftId}), GET /drafts (list from getAllDrafts, sorted newest-first), GET
    /drafts/{id} (one row), GET /drafts/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- DraftingTasks.java declaring three Task<R> constants:
    OUTLINE_DOCUMENT = Task.name("Outline document").description("Structure the prompt into
      2-5 logical sections with headings and brief descriptions").resultConformsTo(Outline.class);
    DRAFT_DOCUMENT = Task.name("Draft document").description("Write body paragraphs for each
      outline section and gather supporting citations").resultConformsTo(Draft.class);
    REFINE_DOCUMENT = Task.name("Refine document").description("Polish each section for
      clarity, add citation markers, and produce the final RefinedDocument")
      .resultConformsTo(RefinedDocument.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- OutlineTools.java — @FunctionTool structurePrompt(String prompt) -> List<Section> reading
  from src/main/resources/sample-data/templates/*.json keyed by prompt topic slug; @FunctionTool
  fetchSectionTemplates(String domain) -> List<String> reading template heading suggestions
  from the matching file.

- DraftTools.java — @FunctionTool writeSection(String sectionId, String context) ->
  SectionDraft (body composed from sample-data citations for the topic); @FunctionTool
  gatherCitations(String topic) -> List<Citation> reading from
  src/main/resources/sample-data/citations/*.json keyed by topic slug.

- RefineTools.java — @FunctionTool polishSection(String sectionDraftJson) -> RefinedSection
  (applies a deterministic cleanup pass on the body text: joins sentences, removes duplicate
  words, normalises punctuation); @FunctionTool formatCitation(String citationJson) -> String
  (APA-style formatted string for the bibliography).

- OutputGuardrail.java — implements the before-agent-response hook on the REFINE task. After
  the agent returns a RefinedDocument, checks: (1) sections.size() >= 2, (2) total word count
  >= 150, (3) bibliography.size() >= 1. On any failure, returns
  Guardrail.reject("quality-violation: <check> failed — <detail>"). On reject ALSO calls
  DraftEntity.recordGuardrailRejection(check, reason) so the rejection is visible in the UI's
  rejection-log strip. On accept, the call proceeds.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: RefinedDocument, Draft,
  Outline. Outputs: QualityResult with score and rationale. Four checks, one point per check
  satisfied, starting from a base of 1: outline coverage (every Outline.section has a matching
  SectionDraft), draft coverage (every SectionDraft is incorporated into a RefinedSection),
  citation provenance (every bibliography entry appeared in Draft.citations), and structural
  parity (refinedDocument.sections.size() == outline.sections.size()). Score range 1-5.
  Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9999 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/prompts.jsonl with 5 seeded prompt lines covering the three
  surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/templates/*.json — three files keyed by seeded prompt topic,
  each carrying 3-5 Section entries with deterministic headings and descriptions.

- src/main/resources/sample-data/citations/*.json — three files keyed by seeded prompt topic,
  each carrying 4-8 Citation entries with label, url, and snippet fields.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (prompts are
  topic-level, not person-level), decisions.authority_level = recommend-only (the refined
  document is advisory), oversight.human_in_loop = true (a human reads the document before
  publishing it), operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "hallucinated-citation", "missing-section-coverage",
  "output-below-quality-threshold", "structural-parity-failure"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/drafting-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Prompt Chaining Workflow", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of draft cards; right = selected-draft detail with prompt header, outline sections
  list, draft body panels, refined document text, quality score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Prompt Chaining Workflow</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(draftId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    outline-document.json — 5 Outline entries, each with 3-5 Section items per seeded
      prompt. Each entry's tool_calls array contains 1-2 calls: 1 structurePrompt(prompt)
      + optionally 1 fetchSectionTemplates(domain) call.
    draft-document.json — 5 Draft entries paired one-to-one with the outline entries,
      each with 3-5 SectionDraft items and 3-6 Citation items, with tool_calls containing
      writeSection (one per section) + gatherCitations in order.
    refine-document.json — 5 RefinedDocument entries paired one-to-one with the draft
      entries. Each carries 3-5 RefinedSection items, tool_calls containing polishSection
      (one per section) + formatCitation (one per citation). Plus 1 deliberately
      QUALITY-FAILING entry whose sections list contains only 1 section (below the
      minimum-2-section threshold) — the OutputGuardrail rejects it; the workflow retries
      refineStep; J2 verifies this. Plus 1 deliberately HALLUCINATED-CITATION entry whose
      bibliography contains a citation absent from the paired Draft.citations — the evalStep
      scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(draftId) helper makes per-draft selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DraftingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DraftingTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (outlineStep
  60s, draftStep 90s, refineStep 90s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the DraftRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: DraftingTasks.java with OUTLINE_DOCUMENT, DRAFT_DOCUMENT, REFINE_DOCUMENT
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9999 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and arrow
  labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (DraftingAgent). The
  on-decision eval is rule-based (QualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each step's tool set is registered on the agent, but
  the before-agent-response guardrail (OutputGuardrail) is the runtime mechanism that
  validates the composite REFINE result. Do NOT conditionally register tools per task — the
  workflow step-chaining enforces step order; the guardrail validates the output.
- Task dependency is carried by typed task results: outlineStep writes Outline onto the
  entity, draftStep reads it and builds the DRAFT task's instruction context from it,
  refineStep reads both. The agent itself is stateless across steps.
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
