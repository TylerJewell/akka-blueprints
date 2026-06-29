# SPEC — pitch-builder

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Pitch Builder.
**One-line pitch:** A user submits a target company name; one `PitchAgent` walks it through three task phases — **RESEARCH** raw deal data, **RUN COMPARABLES** to build a trading and transaction comp table, **DRAFT** a structured pitchbook — with counterparty PII sanitized before the agent sees research outputs and every financial figure validated before the draft is returned to the caller.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a finance-analysis domain. One `PitchAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RESEARCH task's typed output becomes the COMPARABLES task's instruction context; the COMPARABLES task's typed output becomes the DRAFT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **PII sanitizer** runs between the raw tool output and the agent's task-context assembly. Each research tool returns a `RawResearchItem`; before the RESEARCH task's typed result (`ResearchPack`) is written onto the entity and forwarded as the next task's context, the sanitizer scans every `RawResearchItem.metadata` for personal-name, personal-email, and direct-phone patterns and replaces matched text with `[REDACTED]`. The sanitizer uses a configurable pattern registry; its redaction log is appended to `PitchbookEntity` as a `PiiRedactionLogged` event so the audit trail records what was removed and when. The agent never sees the original unredacted value.
- A **`before-agent-response` guardrail** runs immediately after the DRAFT task returns a `Pitchbook` but before the workflow writes `DraftWritten` onto the entity or returns the result to the caller. A deterministic, rule-based `CitationValidator` (no LLM call) checks that every financial figure cited in the pitchbook's sections can be traced to a recorded entry in the `CompsTable`, that every peer company named in a section appears in the `CompsTable.peers` list, and that every valuation multiple cited falls within the `CompsTable.multiples` ranges. If any check fails the guardrail rejects the response, returns a structured `citation-violation` error to the agent loop, and the agent retries within its 4-iteration budget.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the PII sanitizer enforces a data-cleanliness contract at the task boundary, and the response-level guardrail enforces a citation-provenance contract before any client-facing output leaves the service.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **target company name** into the input (or picks one of three seeded targets — `Meridian Packaging Group`, `Ashford Digital Systems`, `Lakeshore Consumer Brands`).
2. The user clicks **Build pitchbook**. The UI POSTs to `/api/pitchbooks` and receives a `pitchbookId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RESEARCHING` — the workflow has started `researchStep` and the agent has been handed the RESEARCH task.
4. Within ~10–20 s the card reaches `RESEARCHED` — the typed `ResearchPack` is visible in the card detail (a table of research items with source, headline, and a redaction-count badge). The agent's RESEARCH task returned; the workflow recorded `TargetResearched` and ran the COMPARABLES task.
5. Within ~10–20 s more the card reaches `COMPS_READY`. The `CompsTable` is visible (peer list + multiples table with EV/EBITDA, EV/Revenue, P/E ranges).
6. Within ~10–20 s more the card reaches `VALIDATED`. The right pane now shows the full typed `Pitchbook` — cover page metadata, executive summary, thesis sections, comparable-company analysis section, and a transaction overview — plus a citation-validation score chip (1–5) and a one-line rationale.
7. The user can submit another target; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PitchbookEndpoint` | `HttpEndpoint` | `/api/pitchbooks/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PitchbookEntity`, `PitchbookView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PitchbookEntity` | `EventSourcedEntity` | Per-pitchbook lifecycle: created → researching → researched → comps-running → comps-ready → drafting → drafted → validated. Source of truth. | `PitchbookEndpoint`, `PitchbookPipelineWorkflow` | `PitchbookView` |
| `PitchbookPipelineWorkflow` | `Workflow` | One workflow per pitchbook. Steps: `researchStep` → `comparablesStep` → `draftStep` → `validationStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `PitchbookEndpoint` after `CREATED` | `PitchAgent`, `PitchbookEntity` |
| `PitchAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `PitchTasks.java`: `RESEARCH_TARGET` → `ResearchPack`, `RUN_COMPARABLES` → `CompsTable`, `DRAFT_PITCHBOOK` → `Pitchbook`. Each task is registered with the phase-appropriate function tools. | invoked by `PitchbookPipelineWorkflow` | returns typed results |
| `ResearchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchFilings(target)` and `fetchHeadline(url)`. Reads from `src/main/resources/sample-data/targets/*.json` for deterministic offline output. | called from RESEARCH task | returns `List<RawResearchItem>` |
| `ComparablesTools` | function-tools class | Implements `selectPeers(sector, targetProfile)` and `buildMultiplesTable(peers)`. Pure in-memory transformations against the sample-data corpus. | called from COMPARABLES task | returns `List<PeerCompany>` / `MultiplesTable` |
| `DraftTools` | function-tools class | Implements `formatSection(thesis, comps)` and `buildCoverPage(target, comps)`. | called from DRAFT task | returns `PitchSection` / `CoverPage` |
| `PiiSanitizer` | plain class (no Akka primitive) | Scans each `RawResearchItem.metadata` field for personal-name, personal-email, and direct-phone patterns using a configurable `PiiPatternRegistry`. Redacts matched text with `[REDACTED]` and appends a `PiiRedactionLogged` event. | runs in `researchStep` after the RESEARCH task returns | redacted `ResearchPack` |
| `CitationValidator` | `before-agent-response` guardrail (registered on `PitchAgent`) | Runs after the DRAFT task returns a `Pitchbook`. Checks that every financial figure cited traces to a `CompsTable` entry, every named peer appears in `CompsTable.peers`, and every multiple cited is within `CompsTable.multiples` ranges. On failure returns a structured `citation-violation` error; on acceptance the workflow writes `DraftWritten`. | every response from the DRAFT task | accept / structured-reject |
| `PitchbookView` | `View` | Read model: one row per pitchbook for the UI. | `PitchbookEntity` events | `PitchbookEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record RawResearchItem(
    String source,
    String url,
    String headline,
    String metadata,   // may contain PII; sanitized before leaving researchStep
    Instant capturedAt
) {}

record ResearchPack(
    String target,
    List<RawResearchItem> items,
    int redactionCount,
    Instant researchedAt
) {}

record PeerCompany(
    String ticker,
    String name,
    String sector,
    double marketCapM   // USD millions
) {}

record MultiplesTable(
    double evEbitdaLow,
    double evEbitdaHigh,
    double evRevenueLow,
    double evRevenueHigh,
    double peRatioLow,
    double peRatioHigh
) {}

record CompsTable(
    List<PeerCompany> peers,
    MultiplesTable multiples,
    Instant builtAt
) {}

record CoverPage(
    String targetName,
    String sector,
    String preparedBy,
    Instant preparedAt
) {}

record PitchSection(
    String sectionId,
    String heading,
    String body,
    List<String> citedTickers,   // MUST each appear in CompsTable.peers[].ticker
    List<String> citedFigures    // each figure MUST trace to CompsTable.multiples
) {}

record Pitchbook(
    CoverPage cover,
    String executiveSummary,
    List<PitchSection> sections,
    Instant draftedAt
) {}

record ValidationResult(
    int score,            // 1..5
    String rationale,
    Instant validatedAt
) {}

record PitchbookRecord(
    String pitchbookId,
    Optional<String> target,
    Optional<ResearchPack> research,
    Optional<CompsTable> comps,
    Optional<Pitchbook> pitchbook,
    Optional<ValidationResult> validation,
    PitchbookStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PitchbookStatus {
    CREATED, RESEARCHING, RESEARCHED,
    COMPS_RUNNING, COMPS_READY,
    DRAFTING, DRAFTED, VALIDATED, FAILED
}
```

Events on `PitchbookEntity`: `PitchbookCreated`, `ResearchStarted`, `TargetResearched`, `CompsStarted`, `ComparablesBuilt`, `DraftStarted`, `DraftWritten`, `ValidationScored`, `PiiRedactionLogged`, `CitationRejected`, `PitchbookFailed`.

Every nullable lifecycle field on the `PitchbookRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/pitchbooks` — body `{ target }` → `{ pitchbookId }`.
- `GET /api/pitchbooks` — list all pitchbooks, newest-first.
- `GET /api/pitchbooks/{id}` — one pitchbook.
- `GET /api/pitchbooks/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Pitch Builder</title>`.

The App UI tab is a two-column layout: a left rail with the live list of pitchbooks (status pill + target name + age) and a right pane with the selected pitchbook's detail — target, research items table with redaction-count badge, comps table (peers + multiples), pitchbook sections, validation score chip, and a citation-rejection log strip if any draft-response rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer**: `PiiSanitizer` runs inside `researchStep`, after the RESEARCH task returns a `Pitchbook`-precursor `ResearchPack` and before the result is written onto `PitchbookEntity` or forwarded to `comparablesStep`. It scans each `RawResearchItem.metadata` field against `PiiPatternRegistry` (configurable list of Java regex patterns; default patterns cover personal names in "LASTNAME, Firstname" format, `@`-delimited personal email addresses, and `+1` or local-format phone numbers). Every matched span is replaced with `[REDACTED]`; the sanitizer increments `redactionCount` and calls `PitchbookEntity.logPiiRedaction(field, patternId, offsetStart, offsetEnd)` so the redaction is in the audit log. The agent's COMPARABLES and DRAFT tasks never see the original unredacted text — it only exists in the raw `RawResearchItem` for the fraction of a millisecond before the sanitizer runs.
- **H1 — `before-agent-response` guardrail (citation gate)**: `CitationValidator` is registered on `PitchAgent` via the agent's guardrail-configuration block, bound to the `before-agent-response` hook. It fires only when the in-flight task is `DRAFT_PITCHBOOK` (the other two tasks do not produce client-facing citations). After the DRAFT task returns a `Pitchbook`, the validator checks three rules: (1) every `PitchSection.citedTickers[i]` value MUST match a `PeerCompany.ticker` in the recorded `CompsTable.peers`; (2) every `PitchSection.citedFigures[j]` value (a decimal string with a unit suffix, e.g. `"12.4x EV/EBITDA"`) MUST be within the range `[CompsTable.multiples.evEbitdaLow, CompsTable.multiples.evEbitdaHigh]` (or the revenue or P/E range for the appropriate figure type); (3) no section cites a peer company not in the recorded `CompsTable.peers` list. On any failure the guardrail returns a structured `citation-violation: <detail>` error to the agent loop and calls `PitchbookEntity.logCitationRejection(sectionId, rule, detail)` for the UI's rejection-log strip. The agent retries within its 4-iteration budget. On success the workflow writes `DraftWritten`.

## 9. Agent prompts

- `PitchAgent` → `prompts/pitch-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded target `Meridian Packaging Group`; within 60 s the pitchbook reaches `VALIDATED` with a non-empty research pack, ≥ 3 peer companies, ≥ 2 sections, and a validation score chip on the card.
2. **J2** — A research item whose `metadata` field contains a personal email address is sanitized; the email does not appear in the recorded `ResearchPack`, in the task context forwarded to COMPARABLES or DRAFT, or anywhere in the final `Pitchbook`. The `PiiRedactionLogged` event is visible in the entity log.
3. **J3** — A draft whose first section cites a ticker absent from the recorded `CompsTable.peers` is rejected by `CitationValidator`; a `CitationRejected` event lands on the entity; the agent retries and produces a corrected draft that cites only recorded peers. The rejection-log strip on the UI card shows the one rejected response.
4. **J4** — A draft citing a valuation multiple outside the `CompsTable.multiples` ranges scores ≤ 2 in validation; the card border highlights red and the rationale names the out-of-range figure.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named pitch-builder demonstrating the sequential-pipeline x finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-finance-analysis-pitch-builder. Java package io.akka.samples.pitchbuilder.
Akka 3.6.0. HTTP port 9114.

Components to wire (exactly):

- 1 AutonomousAgent PitchAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/pitch-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  RESEARCH, COMPARABLES, and DRAFT tool sets are ALL registered on the agent; phase gating is
  handled by the before-agent-response guardrail (CitationValidator) and by workflow step
  ordering, NOT by conditional .tools(...) wiring. The before-agent-response guardrail
  (CitationValidator) is registered on the agent via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow PitchbookPipelineWorkflow per pitchbookId with four steps:
  * researchStep — emits ResearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(PitchAgent.class, "agent-" + pitchbookId).runSingleTask(
      TaskDef.instructions("Target: " + target + "\nPhase: RESEARCH\nUse the filing search
      and headline tools to collect 4-8 research items about this target company.")
        .metadata("pitchbookId", pitchbookId)
        .metadata("phase", "RESEARCH")
        .taskType(PitchTasks.RESEARCH_TARGET)
    ). Reads forTask(taskId).result(RESEARCH_TARGET) to get the raw ResearchPack. Runs
    PiiSanitizer.sanitize(rawPack) to produce a redacted ResearchPack. Writes
    PitchbookEntity.recordResearch(redactedPack). WorkflowSettings.stepTimeout 60s.
  * comparablesStep — emits CompsStarted, then runSingleTask with TaskDef.instructions
    (formatCompsContext(redactedPack, target)) and metadata.phase = "COMPARABLES", taskType
    RUN_COMPARABLES. Writes PitchbookEntity.recordComps(comps). stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(comps, redactedPack, target)) and metadata.phase = "DRAFT", taskType
    DRAFT_PITCHBOOK. The CitationValidator guardrail fires before the result is accepted.
    Writes PitchbookEntity.recordDraft(pitchbook). stepTimeout 60s.
  * validationStep — runs the deterministic CitationValidator final-score over (pitchbook,
    comps) — this is the after-the-fact scoring pass distinct from the per-response guardrail —
    and writes PitchbookEntity.recordValidation(result). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(PitchbookPipelineWorkflow::error). The error step writes
  PitchbookFailed and ends.

- 1 EventSourcedEntity PitchbookEntity (one per pitchbookId). State PitchbookRecord{
  pitchbookId, target: Optional<String>, research: Optional<ResearchPack>,
  comps: Optional<CompsTable>, pitchbook: Optional<Pitchbook>,
  validation: Optional<ValidationResult>, status: PitchbookStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. PitchbookStatus enum: CREATED, RESEARCHING, RESEARCHED,
  COMPS_RUNNING, COMPS_READY, DRAFTING, DRAFTED, VALIDATED, FAILED. Events:
  PitchbookCreated{target}, ResearchStarted, TargetResearched{research}, CompsStarted,
  ComparablesBuilt{comps}, DraftStarted, DraftWritten{pitchbook},
  ValidationScored{validation}, PiiRedactionLogged{field, patternId, offsetStart, offsetEnd},
  CitationRejected{sectionId, rule, detail}, PitchbookFailed{reason}.
  Commands: create, startResearch, recordResearch, startComps, recordComps, startDraft,
  recordDraft, recordValidation, logPiiRedaction, logCitationRejection, fail, getPitchbook.
  emptyState() returns PitchbookRecord.initial("") with all Optional fields as Optional.empty()
  and no commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View PitchbookView with row type PitchbookRow that mirrors PitchbookRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes PitchbookEntity events. ONE
  query getAllPitchbooks: SELECT * AS pitchbooks FROM pitchbook_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PitchbookEndpoint at /api with POST /pitchbooks (body {target}; mints pitchbookId; calls
    PitchbookEntity.create(target); then starts PitchbookPipelineWorkflow with id
    "pipeline-" + pitchbookId; returns {pitchbookId}), GET /pitchbooks (list from
    getAllPitchbooks, sorted newest-first), GET /pitchbooks/{id} (one row), GET
    /pitchbooks/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- PitchTasks.java declaring three Task<R> constants:
    RESEARCH_TARGET = Task.name("Research target").description("Gather raw research items about
      a target company by calling searchFilings and fetchHeadline").resultConformsTo(
      ResearchPack.class);
    RUN_COMPARABLES = Task.name("Run comparables").description("Select peer companies then
      build a multiples table from in-process data").resultConformsTo(CompsTable.class);
    DRAFT_PITCHBOOK = Task.name("Draft pitchbook").description("Compose a Pitchbook whose
      sections cite only peers and multiples from the recorded CompsTable").resultConformsTo(
      Pitchbook.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- PitchPhase.java — enum {RESEARCH, COMPARABLES, DRAFT}. Used in workflow step metadata.

- PiiPatternRegistry.java — holds the default list of java.util.regex.Pattern entries for
  PiiSanitizer: (1) personal-name pattern matching "LASTNAME, Firstname" or "Firstname
  LASTNAME" preceded by a title word like "Contact:", "Director:", "MD:", "Analyst:";
  (2) personal-email pattern matching strings of form word@word.tld; (3) direct-phone pattern
  matching +1 NNNN or local (NNN) NNN-NNNN. Patterns are final static fields; the class is
  instantiable so deployers can subclass and override the list.

- PiiSanitizer.java — stateless utility class. sanitize(ResearchPack raw) iterates each
  RawResearchItem.metadata(), applies each PiiPatternRegistry pattern with Matcher.replaceAll(
  "[REDACTED]"), counts replacements, and returns a new ResearchPack with the sanitized items
  and redactionCount. Each replacement also triggers PitchbookEntity.logPiiRedaction via the
  componentClient so the event is in the audit log.

- ResearchTools.java — @FunctionTool searchFilings(String target) -> List<RawResearchItem>
  reading from src/main/resources/sample-data/targets/*.json keyed by target slug; @FunctionTool
  fetchHeadline(String url) -> String reading from the matching research item's headline.

- ComparablesTools.java — @FunctionTool selectPeers(String sector, String targetProfile) ->
  List<PeerCompany> (select 3-5 peers from the sample-data corpus matching the sector);
  @FunctionTool buildMultiplesTable(List<PeerCompany> peers) -> MultiplesTable (compute
  EV/EBITDA, EV/Revenue, P/E ranges from the peers' stored multiples).

- DraftTools.java — @FunctionTool formatSection(String thesis, CompsTable comps) -> PitchSection
  (heading from thesis, body citing named peers and multiples from the comps table); @FunctionTool
  buildCoverPage(String target, CompsTable comps) -> CoverPage (target name, sector derived from
  comps peer list, preparedBy = "Pitch Builder", preparedAt = Instant.now()).

- CitationValidator.java — implements the before-agent-response hook. On DRAFT_PITCHBOOK task
  responses, reads each PitchSection.citedTickers and verifies each ticker appears in
  CompsTable.peers[].ticker. Reads each PitchSection.citedFigures and parses the numeric value
  and multiple type (EV/EBITDA / EV/Revenue / P/E) and verifies it is within the corresponding
  CompsTable.multiples range (±0.1x tolerance for rounding). Verifies no peer company name
  referenced in the section body is absent from CompsTable.peers[].name. On any failure calls
  PitchbookEntity.logCitationRejection(sectionId, rule, detail) and returns
  Guardrail.reject("citation-violation: " + detail). On accept, the draft proceeds. Also used
  in validationStep for the final scoring pass: four checks, one point each on a base of 1 —
  ticker coverage, figure provenance, peer name coverage, section count ≥ 1. Score range 1–5.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9114 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/targets.jsonl with 5 seeded target-company lines covering
  the three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/targets/*.json — three files keyed by target slug, each
  carrying 6-10 RawResearchItem entries. One entry per file contains a personal email address
  in the metadata field so PiiSanitizer fires on every seeded target. Each file also carries
  the matching peer list and multiples for ComparablesTools.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, H1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — sample domain.

- risk-survey.yaml at the project root with data.data_classes.pii = true (research inputs
  may reference counterparty contact names), decisions.authority_level = recommend-only (the
  pitchbook is advisory material reviewed by a human analyst before any client communication),
  oversight.human_in_loop = true (analyst reviews the pitchbook before sending to client),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "hallucinated-peer", "out-of-range-multiple",
  "pii-in-draft", "missing-comps-coverage"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/pitch-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Pitch Builder", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of pitchbook cards; right = selected-pitchbook detail with target header, research
  items table with redaction-count badge, comps table, pitchbook sections, validation-score
  chip, citation-rejection log strip). Browser title exactly:
  <title>Akka Sample: Pitch Builder</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(pitchbookId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    research-target.json — 6 ResearchPack entries, each with 5-8 RawResearchItem items per
      seeded target. Each entry includes at least one RawResearchItem whose metadata field
      contains a personal email address in the form "Contact: analyst.name@firm.example.com"
      — the PiiSanitizer must redact it. tool_calls array: 1 searchFilings(target) + 1-3
      fetchHeadline(url) calls.
    run-comparables.json — 6 CompsTable entries each paired with the matching target and
      containing 3-5 PeerCompany entries and a realistic MultiplesTable. tool_calls:
      selectPeers + buildMultiplesTable.
    draft-pitchbook.json — 6 Pitchbook entries paired one-to-one. Each carries 2-3
      PitchSection items with citedTickers and citedFigures referencing the paired CompsTable.
      Plus 1 deliberately HALLUCINATED-PEER entry whose first section cites a ticker absent
      from the recorded CompsTable — the CitationValidator guardrail rejects it; the agent
      retries; the second iteration produces a corrected draft. The mock should select the
      violating entry on the FIRST iteration of every 3rd pitchbook (modulo seed) so J3 is
      reproducible.
- A MockModelProvider.seedFor(pitchbookId) helper makes per-pitchbook selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PitchAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion PitchTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (researchStep
  60s, comparablesStep 60s, draftStep 60s, validationStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the PitchbookRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: PitchTasks.java with RESEARCH_TARGET, RUN_COMPARABLES, DRAFT_PITCHBOOK constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9114 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (PitchAgent). The
  citation validation scoring is rule-based (CitationValidator.java) and does NOT make an
  LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, and
  phase ordering is enforced by the workflow step sequence — RESEARCH runs first, only its
  typed result is forwarded to COMPARABLES, only the COMPARABLES result is forwarded to DRAFT.
  The CitationValidator governs the response boundary, not tool access.
- Task dependency is carried by typed task results: researchStep writes ResearchPack onto the
  entity (after PII sanitization), comparablesStep reads it and builds the COMPARABLES task's
  instruction context from it, draftStep reads both. The agent itself is stateless across
  phases.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
