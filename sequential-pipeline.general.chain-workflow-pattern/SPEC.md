# SPEC — chain-workflow

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Chain Workflow.
**One-line pitch:** A user submits raw content; one `TransformAgent` walks it through three transformation stages — **EXTRACT** a typed structure from the raw input, **REFINE** the structure into polished content, **FORMAT** it into a complete structured `Document` — with each stage's output validated before passing downstream and each stage's LLM call receiving only its own typed input.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern implemented as prompt chaining in a general-purpose document-transformation domain. One `TransformAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **typed stage handoff**: the EXTRACT task's typed output (`ExtractedStructure`) becomes the REFINE task's instruction context; the REFINE task's typed output (`RefinedContent`) becomes the FORMAT task's instruction context. The agent never holds all three stages in one conversation — each stage runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- An **`after-llm-response` guardrail** sits between the agent's LLM response and the workflow recording it. After each stage completes, `OutputGuardrail` inspects the typed result: for EXTRACT, it checks that at least one `Fact` was produced and all required fields are populated; for REFINE, it checks that every `RefinedFact` traces to a source `Fact.factId` and the overall word count is within bounds; for FORMAT, it checks that every `Section` cites at least one `RefinedFact` and the `Document.title` is non-empty. A result that fails validation is returned as a structured rejection to the agent loop so the task can retry inside its iteration budget. The rejection is recorded on `DocumentEntity` as `GuardrailRejected{stage, failedField, reason}` for visibility.
- An **`on-decision-eval`** runs immediately after `DocumentFormatted` lands, as `evalStep` inside the workflow. A deterministic, rule-based `QualityScorer` (no LLM call — the eval is rule-based on purpose) checks fact coverage (every extracted fact is referenced in the refined content), citation completeness (every section cites at least one fact), structural parity (section count equals the number of refined content blocks), and title presence.

The blueprint shows that prompt chaining is more than a sequence of LLM calls — the typed stage boundary is the right cut to validate output quality, enforce the dependency contract, and keep each call's context tight.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types **raw content** into the input (or picks one of three seeded inputs — `Product release notes draft`, `Technical specification excerpt`, `Research abstract summary`).
2. The user clicks **Run chain**. The UI POSTs to `/api/documents` and receives a `documentId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `EXTRACTING` — the workflow has started `extractStep` and the agent has been handed the EXTRACT task.
4. Within ~10–20 s the card reaches `REFINING` — the typed `ExtractedStructure` is visible in the card detail (a small table of facts with id, text, and category). The agent's EXTRACT task returned; the workflow validated it via `OutputGuardrail` and recorded `StructureExtracted`; then the workflow ran the REFINE task.
5. Within ~10–20 s more the card reaches `FORMATTING`. The `RefinedContent` is visible (refined-fact list with source references and polish notes).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `Document` — title, summary, per-section `Section` with heading, body, and cited facts — plus a quality score chip (1–5) and a one-line rationale.
7. The user can submit another piece of content; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DocumentEndpoint` | `HttpEndpoint` | `/api/documents/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `DocumentEntity`, `DocumentView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DocumentEntity` | `EventSourcedEntity` | Per-document lifecycle: created → extracting → extracted → refining → refined → formatting → formatted → evaluated. Source of truth. | `DocumentEndpoint`, `TransformPipelineWorkflow` | `DocumentView` |
| `TransformPipelineWorkflow` | `Workflow` | One workflow per document. Steps: `extractStep` → `refineStep` → `formatStep` → `evalStep`. Each step runs one task on the agent, validates the typed result via `OutputGuardrail`, writes the corresponding event onto the entity, then advances. | started by `DocumentEndpoint` after `CREATED` | `TransformAgent`, `DocumentEntity` |
| `TransformAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `TransformTasks.java`: `EXTRACT_STRUCTURE` → `ExtractedStructure`, `REFINE_CONTENT` → `RefinedContent`, `FORMAT_DOCUMENT` → `Document`. Each task is registered with the stage-appropriate function tools. | invoked by `TransformPipelineWorkflow` | returns typed results |
| `ExtractTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `parseInput(rawText)` and `classifyFacts(rawText)`. Pure in-process transformations over the submitted text. | called from EXTRACT task | returns `List<Fact>` |
| `RefineTools` | function-tools class | Implements `polishFact(fact)` and `mergeDuplicates(facts)`. Pure in-memory text transformations. | called from REFINE task | returns `List<RefinedFact>` |
| `FormatTools` | function-tools class | Implements `buildSection(refinedFacts, groupLabel)` and `composeSummary(sections)`. | called from FORMAT task | returns `Section` / `String` |
| `OutputGuardrail` | `after-llm-response` guardrail (registered on `TransformAgent`) | Validates the typed result returned by each LLM call. Checks required fields, cross-stage referential integrity, and size bounds. On rejection returns a structured error to the agent loop and records `GuardrailRejected` on the entity. | every task completion on every stage | accept / structured-reject |
| `QualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `Document`, `RefinedContent`, `ExtractedStructure`. Output: `QualityResult{score, rationale}`. | called from `evalStep` | returns score |
| `DocumentView` | `View` | Read model: one row per document for the UI. | `DocumentEntity` events | `DocumentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Fact(String factId, String text, String category, String sourceSpan) {}

record ExtractedStructure(List<Fact> facts, Instant extractedAt) {}

record RefinedFact(String refinedId, String text, String sourceFactId, String polishNote) {}

record RefinedContent(List<RefinedFact> refinedFacts, Instant refinedAt) {}

record CitedFact(String refinedId, String text) {}

record Section(String sectionId, String heading, String body, List<CitedFact> citations) {}

record Document(
    String title,
    String summary,
    List<Section> sections,
    Instant formattedAt
) {}

record QualityResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record DocumentRecord(
    String documentId,
    Optional<String> rawInput,
    Optional<ExtractedStructure> extracted,
    Optional<RefinedContent> refined,
    Optional<Document> document,
    Optional<QualityResult> quality,
    DocumentStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DocumentStatus {
    CREATED, EXTRACTING, EXTRACTED, REFINING, REFINED,
    FORMATTING, FORMATTED, EVALUATED, FAILED
}
```

Events on `DocumentEntity`: `DocumentCreated`, `ExtractStarted`, `StructureExtracted`, `RefineStarted`, `ContentRefined`, `FormatStarted`, `DocumentFormatted`, `QualityScored`, `GuardrailRejected`, `DocumentFailed`.

Every nullable lifecycle field on the `DocumentRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/documents` — body `{ rawInput }` → `{ documentId }`.
- `GET /api/documents` — list all documents, newest-first.
- `GET /api/documents/{id}` — one document.
- `GET /api/documents/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Chain Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of documents (status pill + input label + age) and a right pane with the selected document's detail — raw input, extracted facts table, refined content list, formatted document sections, quality score chip, and a guardrail-rejection log strip if any output-validation rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `after-llm-response` guardrail (output validation)**: `OutputGuardrail` is registered on `TransformAgent` and runs after every task completion, before the workflow records the result. It reads the typed output and the current stage (encoded in the in-flight `TaskDef.metadata` as `stage = EXTRACT / REFINE / FORMAT`). The accept matrix is stage-specific: EXTRACT output requires `facts.size() >= 1` and every `Fact.factId` non-empty; REFINE output requires every `RefinedFact.sourceFactId` to match a `Fact.factId` in the upstream `ExtractedStructure`; FORMAT output requires every `Section.citations` list non-empty and `document.title` non-empty. On reject, the guardrail returns a structured `output-validation-failure` error to the agent loop and the workflow records `GuardrailRejected{stage, failedField, reason}` for visibility. The agent loop retries within its 4-iteration budget.
- **E1 — `on-decision-eval`**: runs immediately after `DocumentFormatted` lands, as `evalStep` inside the workflow. `QualityScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every extracted fact must be referenced in the refined content (fact coverage), every section must cite at least one `RefinedFact` (citation completeness), the section count must equal the number of top-level content blocks in the `RefinedContent` (structural parity), and the `Document.title` must be non-empty (title presence). Emits `QualityScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `TransformAgent` → `prompts/transform-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, call only the tools whose names match that stage's tool set, treat each task's typed input as the entire context for that stage, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded input `Product release notes draft`; within 60 s the document reaches `EVALUATED` with non-empty extracted facts, ≥ 2 refined facts, ≥ 2 sections, and a quality score chip on the card.
2. **J2** — The agent's first iteration on a document returns an EXTRACT output with an empty `facts` list (mock LLM path). `OutputGuardrail` rejects it; a `GuardrailRejected` event lands on the entity; the agent retries; the document eventually completes correctly. The UI's rejection-log strip shows the one rejected output.
3. **J3** — A document whose mock-LLM trajectory produces a `Section` with no citations is scored 1 with a rationale naming the uncited section; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-document trace (logged at `INFO`); the EXTRACT task's log shows only EXTRACT-tool calls, the REFINE task's log shows only REFINE-tool calls, the FORMAT task's log shows only FORMAT-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named chain-workflow demonstrating the sequential-pipeline x general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-general-chain-workflow-pattern. Java package io.akka.samples.chainworkflow.
Akka 3.6.0. HTTP port 9974.

Components to wire (exactly):

- 1 AutonomousAgent TransformAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/transform-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  EXTRACT, REFINE, and FORMAT tool sets are ALL registered on the agent. The after-llm-response
  guardrail (OutputGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow TransformPipelineWorkflow per documentId with four steps:
  * extractStep — emits ExtractStarted on the entity, then calls componentClient
    .forAutonomousAgent(TransformAgent.class, "agent-" + documentId).runSingleTask(
      TaskDef.instructions("Raw input:\n" + rawInput + "\nStage: EXTRACT\nParse the raw text
      into typed Facts.")
        .metadata("documentId", documentId)
        .metadata("stage", "EXTRACT")
        .taskType(TransformTasks.EXTRACT_STRUCTURE)
    ). Reads forTask(taskId).result(EXTRACT_STRUCTURE) to get ExtractedStructure. Writes
    DocumentEntity.recordExtracted(extracted). WorkflowSettings.stepTimeout 60s.
  * refineStep — emits RefineStarted, then runSingleTask with TaskDef.instructions
    (formatRefineContext(extracted, rawInput)) and metadata.stage = "REFINE", taskType
    REFINE_CONTENT. Writes DocumentEntity.recordRefined(refined). stepTimeout 60s.
  * formatStep — emits FormatStarted, then runSingleTask with TaskDef.instructions
    (formatDocumentContext(refined, extracted, rawInput)) and metadata.stage = "FORMAT",
    taskType FORMAT_DOCUMENT. Writes DocumentEntity.recordDocument(document). stepTimeout 60s.
  * evalStep — runs the deterministic QualityScorer over (document, refined, extracted)
    and writes DocumentEntity.recordQuality(quality). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(TransformPipelineWorkflow::error). The error step writes
  DocumentFailed and ends.

- 1 EventSourcedEntity DocumentEntity (one per documentId). State DocumentRecord{documentId,
  rawInput: Optional<String>, extracted: Optional<ExtractedStructure>,
  refined: Optional<RefinedContent>, document: Optional<Document>,
  quality: Optional<QualityResult>, status: DocumentStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. DocumentStatus enum: CREATED, EXTRACTING, EXTRACTED,
  REFINING, REFINED, FORMATTING, FORMATTED, EVALUATED, FAILED. Events: DocumentCreated{rawInput},
  ExtractStarted, StructureExtracted{extracted}, RefineStarted, ContentRefined{refined},
  FormatStarted, DocumentFormatted{document}, QualityScored{quality},
  GuardrailRejected{stage, failedField, reason}, DocumentFailed{reason}. Commands: create,
  startExtract, recordExtracted, startRefine, recordRefined, startFormat, recordDocument,
  recordQuality, recordGuardrailRejection, fail, getDocument. emptyState() returns
  DocumentRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View DocumentView with row type DocumentRow that mirrors DocumentRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes DocumentEntity events. ONE
  query getAllDocuments: SELECT * AS documents FROM document_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * DocumentEndpoint at /api with POST /documents (body {rawInput}; mints documentId; calls
    DocumentEntity.create(rawInput); then starts TransformPipelineWorkflow with id
    "chain-" + documentId; returns {documentId}), GET /documents (list from
    getAllDocuments, sorted newest-first), GET /documents/{id} (one row), GET
    /documents/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- TransformTasks.java declaring three Task<R> constants:
    EXTRACT_STRUCTURE = Task.name("Extract structure").description("Parse raw input text into
      typed Facts using parseInput and classifyFacts").resultConformsTo(ExtractedStructure.class);
    REFINE_CONTENT = Task.name("Refine content").description("Polish and deduplicate facts
      into RefinedFacts using polishFact and mergeDuplicates").resultConformsTo(
      RefinedContent.class);
    FORMAT_DOCUMENT = Task.name("Format document").description("Compose a Document whose
      Sections group the refined facts").resultConformsTo(Document.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Stage.java — enum {EXTRACT, REFINE, FORMAT}. The OutputGuardrail reads the in-flight
  task's stage from the TaskDef metadata to apply the correct validation matrix.

- ExtractTools.java — @FunctionTool parseInput(String rawText) -> List<Fact> splitting the
  text into fact-sized units; @FunctionTool classifyFacts(List<Fact>) -> List<Fact> assigning
  each fact a category (one of: "feature", "limitation", "requirement", "context").

- RefineTools.java — @FunctionTool polishFact(Fact) -> RefinedFact rewriting the fact text
  for clarity and attaching the source factId; @FunctionTool mergeDuplicates(
  List<RefinedFact>) -> List<RefinedFact> collapsing facts with > 80% text overlap,
  preserving all sourceFactIds.

- FormatTools.java — @FunctionTool buildSection(List<RefinedFact>, String groupLabel) ->
  Section grouping refined facts under a heading and body; @FunctionTool composeSummary(
  List<Section>) -> String producing a 2–3 sentence document summary.

- OutputGuardrail.java — implements the after-llm-response hook. Reads the current stage
  from the TaskDef metadata (carried as "stage"), reads the typed result returned by the
  agent, applies the accept matrix from Section 8, and either passes or returns
  Guardrail.reject("output-validation-failure: <field> <reason>"). On reject ALSO calls
  DocumentEntity.recordGuardrailRejection(stage, failedField, reason) so the rejection is
  visible in the UI's rejection-log strip and in the audit log.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: Document,
  RefinedContent, ExtractedStructure. Outputs: QualityResult with score and rationale.
  Four checks, one point per check satisfied, starting from a base of 1: fact coverage
  (every Fact.factId is referenced by at least one RefinedFact.sourceFactId), citation
  completeness (every Section.citations list non-empty), structural parity
  (sections.size() == number of distinct RefinedFact groups as clustered by the FORMAT
  task), and title presence (document.title non-empty). Score range 1-5. Rationale names
  the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9974 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/inputs.jsonl with 5 seeded raw-input lines covering the
  three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/inputs/*.json — three files keyed by seeded input label,
  each carrying deterministic Fact entries so ExtractTools.parseInput returns the same list
  across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (raw input is
  user-supplied text, not person-level), decisions.authority_level = recommend-only (the
  document is advisory), oversight.human_in_loop = true (a human reviews the document
  before acting on it), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "empty-extract-output",
  "uncited-section", "orphaned-fact", "output-validation-failure"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/transform-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Chain Workflow", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of document cards; right = selected-document detail with raw input header,
  extracted facts table, refined content list, formatted document sections, quality score
  chip, rejection-log strip). Browser title exactly: <title>Akka Sample: Chain
  Workflow</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(documentId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    extract-structure.json — 6 ExtractedStructure entries, each with 3-6 Fact items.
      Each entry's tool_calls array contains 2 calls: parseInput(rawText) + classifyFacts(
      facts). Plus 1 deliberately INVALID entry whose returned ExtractedStructure has an
      empty facts list — the guardrail rejects it, the mock then falls through to a normal
      extract sequence. The mock should select the invalid entry on the FIRST iteration
      of every 3rd document (modulo seed) so J2 is reproducible.
    refine-content.json — 6 RefinedContent entries paired one-to-one with the extract
      entries, each with 3-6 RefinedFact items, with tool_calls containing polishFact
      (one per fact) + mergeDuplicates in order.
    format-document.json — 6 Document entries paired one-to-one. Each carries 2-4
      Section items grouping the refined facts, tool_calls containing buildSection
      (one per group) + composeSummary. Plus 1 deliberately UNCITED-SECTION entry whose
      first Section has an empty citations list — the QualityScorer scores it 1; J3
      verifies this.
- A MockModelProvider.seedFor(documentId) helper makes per-document selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TransformAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TransformTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (extractStep
  60s, refineStep 60s, formatStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the DocumentRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: TransformTasks.java with EXTRACT_STRUCTURE, REFINE_CONTENT, FORMAT_DOCUMENT
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9974 declared explicitly in application.conf's
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
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (TransformAgent). The
  on-decision eval is rule-based (QualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each stage's tool set is registered on the agent, and
  the after-llm-response guardrail (OutputGuardrail) validates the stage's typed output
  before the workflow records it. The guardrail is the gate; do not conditionally register
  tools per task.
- Task dependency is carried by typed task results: extractStep writes ExtractedStructure
  onto the entity, refineStep reads it and builds the REFINE task's instruction context from
  it, formatStep reads both. The agent itself is stateless across stages.
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
