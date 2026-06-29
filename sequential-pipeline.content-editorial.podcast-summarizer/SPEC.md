# SPEC — podcast-summarizer

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Podcast Transcript Summarizer.
**One-line pitch:** A user submits a podcast episode transcript; one `TranscriptAgent` walks it through three task phases — **EXTRACT** key quotes, **SEGMENT** them into topic clusters, **SUMMARIZE** each cluster into a structured `EpisodeSummary` — with each phase gated on the prior phase's recorded output and each phase's tools rejected when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a content-editorial domain. One `TranscriptAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the EXTRACT task's typed output becomes the SEGMENT task's instruction context; the SEGMENT task's typed output becomes the SUMMARIZE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`EXTRACT` / `SEGMENT` / `SUMMARIZE`) and the current `EpisodeEntity` status. A SUMMARIZE-phase tool called while the entity has not yet recorded `TopicsSegmented` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. The same hook enforces the deeper pipeline property: a phase's tool set is unreachable until its preconditions hold.
- An **`on-decision-eval`** runs immediately after `SummaryWritten` lands, as `evalStep` inside the workflow. A deterministic, rule-based `FidelityScorer` (no LLM call — the eval is rule-based on purpose, so the same summary always scores the same) checks that every topic segment has a matching summary section, that every summary section cites at least one supporting quote, and that no cited quote was absent from the extracted `QuoteSet`.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and the per-phase tool isolation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **transcript** or **episode title** into the input (or picks one of three seeded episodes — `The Founder's Dilemma with Reid Hoffman`, `Deep Cuts: Postgres internals with Bruce Momjian`, `AI and the Law with Gary Marcus`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/episodes` and receives an `episodeId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `EXTRACTING` — the workflow has started `extractStep` and the agent has been handed the EXTRACT task.
4. Within ~10–20 s the card reaches `SEGMENTING` — the typed `QuoteSet` is visible in the card detail (a small table of quotes with speaker, timestamp, and text preview). The agent's EXTRACT task returned; the workflow recorded `QuotesExtracted` and ran the SEGMENT task.
5. Within ~10–20 s more the card reaches `SUMMARIZING`. The `Segmentation` is visible (topic cluster list + quote assignments per cluster).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `EpisodeSummary` — episode title, overall takeaway, per-topic `SummarySection` with body paragraphs and a `quotes` list — plus an eval score chip (1–5) and a one-line rationale.
7. The user can submit another episode; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EpisodeEndpoint` | `HttpEndpoint` | `/api/episodes/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `EpisodeEntity`, `EpisodeView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `EpisodeEntity` | `EventSourcedEntity` | Per-episode lifecycle: created → extracting → extracted → segmenting → segmented → summarizing → summarized → evaluated. Source of truth. | `EpisodeEndpoint`, `SummaryPipelineWorkflow` | `EpisodeView` |
| `SummaryPipelineWorkflow` | `Workflow` | One workflow per episode. Steps: `extractStep` → `segmentStep` → `summarizeStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `EpisodeEndpoint` after `CREATED` | `TranscriptAgent`, `EpisodeEntity` |
| `TranscriptAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `TranscriptTasks.java`: `EXTRACT_QUOTES` → `QuoteSet`, `SEGMENT_TOPICS` → `Segmentation`, `WRITE_SUMMARY` → `EpisodeSummary`. Each task is registered with the phase-appropriate function tools. | invoked by `SummaryPipelineWorkflow` | returns typed results |
| `ExtractTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `scanTranscript(transcript)` and `fetchSpeakerTurn(turnId)`. Reads from `src/main/resources/sample-data/transcripts/*.json` for deterministic offline output. | called from EXTRACT task | returns `List<Quote>` |
| `SegmentTools` | function-tools class | Implements `clusterQuotes(quotes)` and `labelCluster(clusterQuotes)`. Pure in-memory transformations. | called from SEGMENT task | returns `List<TopicCluster>` |
| `SummarizeTools` | function-tools class | Implements `draftSection(cluster, quotes)` and `writeOverallTakeaway(sections)`. | called from SUMMARIZE task | returns `SummarySection` / `String` |
| `PhaseGuardrail` | `before-tool-call` guardrail (registered on `TranscriptAgent`) | Reads the in-flight task's declared phase and the current `EpisodeEntity` status. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `FidelityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `EpisodeSummary`, `Segmentation`, `QuoteSet`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `EpisodeView` | `View` | Read model: one row per episode for the UI. | `EpisodeEntity` events | `EpisodeEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Quote(String quoteId, String speaker, String text, String timestampLabel) {}

record QuoteSet(List<Quote> quotes, String episodeTitle, Instant extractedAt) {}

record TopicCluster(String clusterId, String topicLabel, List<String> quoteIds) {}

record Segmentation(List<TopicCluster> clusters, Instant segmentedAt) {}

record SummarySection(
    String clusterId,
    String heading,
    String body,
    List<String> supportingQuoteIds
) {}

record EpisodeSummary(
    String episodeTitle,
    String overallTakeaway,
    List<SummarySection> sections,
    Instant writtenAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record EpisodeRecord(
    String episodeId,
    Optional<String> episodeTitle,
    Optional<String> transcriptText,
    Optional<QuoteSet> quotes,
    Optional<Segmentation> segmentation,
    Optional<EpisodeSummary> summary,
    Optional<EvalResult> eval,
    EpisodeStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EpisodeStatus {
    CREATED, EXTRACTING, EXTRACTED, SEGMENTING, SEGMENTED,
    SUMMARIZING, SUMMARIZED, EVALUATED, FAILED
}
```

Events on `EpisodeEntity`: `EpisodeCreated`, `ExtractStarted`, `QuotesExtracted`, `SegmentStarted`, `TopicsSegmented`, `SummarizeStarted`, `SummaryWritten`, `EvaluationScored`, `GuardrailRejected`, `EpisodeFailed`.

Every nullable lifecycle field on the `EpisodeRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/episodes` — body `{ episodeTitle, transcriptText }` → `{ episodeId }`.
- `GET /api/episodes` — list all episodes, newest-first.
- `GET /api/episodes/{id}` — one episode.
- `GET /api/episodes/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Podcast Transcript Summarizer</title>`.

The App UI tab is a two-column layout: a left rail with the live list of episodes (status pill + episode title + age) and a right pane with the selected episode's detail — title, extracted quotes table, topic clusters with quote assignments, summary sections, eval score chip, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (phase-gate)**: `PhaseGuardrail` is registered on `TranscriptAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.EXTRACT`, `Phase.SEGMENT`, `Phase.SUMMARIZE`) and the current `EpisodeEntity.status` for the episode the task is bound to. The accept rule is precise: `EXTRACT` tools require `status ∈ {CREATED, EXTRACTING}`; `SEGMENT` tools require `status ∈ {EXTRACTED, SEGMENTING}` AND `quotes.isPresent()`; `SUMMARIZE` tools require `status ∈ {SEGMENTED, SUMMARIZING}` AND `segmentation.isPresent()`. On reject, the guardrail returns a structured `phase-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility. The agent loop retries within its 4-iteration budget.
- **E1 — `on-decision-eval`**: runs immediately after `SummaryWritten` lands, as `evalStep` inside the workflow. `FidelityScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every topic cluster must have a matching summary section (cluster coverage), every summary section must cite at least one supporting quote id (quote attribution), every cited quote id must appear in the recorded `QuoteSet` (no fabricated quotes), and the section count must equal the cluster count (no silent expansion or collapse). Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `TranscriptAgent` → `prompts/transcript-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded episode `Deep Cuts: Postgres internals with Bruce Momjian`; within 60 s the episode reaches `EVALUATED` with non-empty quotes, ≥ 2 clusters, ≥ 2 sections, and an eval score chip on the card.
2. **J2** — The agent's first iteration on an episode calls a SUMMARIZE-phase tool (`draftSection`) before `TopicsSegmented` has been recorded (mock LLM path). `PhaseGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the episode eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — An episode whose mock-LLM trajectory produces a section citing a quote id absent from the recorded `QuoteSet` is scored 1 with a rationale naming the fabricated quote id; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-episode trace (logged at `INFO`); the EXTRACT task's log shows only EXTRACT-tool calls, the SEGMENT task's log shows only SEGMENT-tool calls, the SUMMARIZE task's log shows only SUMMARIZE-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named podcast-summarizer demonstrating the sequential-pipeline x content-editorial
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-content-editorial-podcast-summarizer. Java package
io.akka.samples.podcasttranscriptagent. Akka 3.6.0. HTTP port 9821.

Components to wire (exactly):

- 1 AutonomousAgent TranscriptAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/transcript-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  EXTRACT, SEGMENT, and SUMMARIZE tool sets are ALL registered on the agent; phase gating is
  the job of PhaseGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (PhaseGuardrail) is registered on the agent via the agent's guardrail-configuration
  block. On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow SummaryPipelineWorkflow per episodeId with four steps:
  * extractStep — emits ExtractStarted on the entity, then calls componentClient
    .forAutonomousAgent(TranscriptAgent.class, "agent-" + episodeId).runSingleTask(
      TaskDef.instructions("Episode: " + episodeTitle + "\nTranscript:\n" + transcriptText +
      "\nPhase: EXTRACT\nUse the scan and fetch tools to extract 6-12 representative quotes.")
        .metadata("episodeId", episodeId)
        .metadata("phase", "EXTRACT")
        .taskType(TranscriptTasks.EXTRACT_QUOTES)
    ). Reads forTask(taskId).result(EXTRACT_QUOTES) to get QuoteSet. Writes
    EpisodeEntity.recordQuotes(quoteSet). WorkflowSettings.stepTimeout 60s.
  * segmentStep — emits SegmentStarted, then runSingleTask with TaskDef.instructions
    (formatSegmentContext(quoteSet, episodeTitle)) and metadata.phase = "SEGMENT", taskType
    SEGMENT_TOPICS. Writes EpisodeEntity.recordSegmentation(segmentation). stepTimeout 60s.
  * summarizeStep — emits SummarizeStarted, then runSingleTask with TaskDef.instructions
    (formatSummarizeContext(segmentation, quoteSet, episodeTitle)) and metadata.phase =
    "SUMMARIZE", taskType WRITE_SUMMARY. Writes EpisodeEntity.recordSummary(summary).
    stepTimeout 60s.
  * evalStep — runs the deterministic FidelityScorer over (summary, segmentation, quoteSet)
    and writes EpisodeEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(SummaryPipelineWorkflow::error). The error step writes
  EpisodeFailed and ends.

- 1 EventSourcedEntity EpisodeEntity (one per episodeId). State EpisodeRecord{episodeId,
  episodeTitle: Optional<String>, transcriptText: Optional<String>,
  quotes: Optional<QuoteSet>, segmentation: Optional<Segmentation>,
  summary: Optional<EpisodeSummary>, eval: Optional<EvalResult>, status: EpisodeStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. EpisodeStatus enum: CREATED,
  EXTRACTING, EXTRACTED, SEGMENTING, SEGMENTED, SUMMARIZING, SUMMARIZED, EVALUATED, FAILED.
  Events: EpisodeCreated{episodeTitle, transcriptText}, ExtractStarted, QuotesExtracted{quotes},
  SegmentStarted, TopicsSegmented{segmentation}, SummarizeStarted, SummaryWritten{summary},
  EvaluationScored{eval}, GuardrailRejected{phase, tool, reason}, EpisodeFailed{reason}.
  Commands: create, startExtract, recordQuotes, startSegment, recordSegmentation,
  startSummarize, recordSummary, recordEvaluation, recordGuardrailRejection, fail, getEpisode.
  emptyState() returns EpisodeRecord.initial("") with all Optional fields as Optional.empty()
  and no commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View EpisodeView with row type EpisodeRow that mirrors EpisodeRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes EpisodeEntity events. ONE
  query getAllEpisodes: SELECT * AS episodes FROM episode_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * EpisodeEndpoint at /api with POST /episodes (body {episodeTitle, transcriptText}; mints
    episodeId; calls EpisodeEntity.create(episodeTitle, transcriptText); then starts
    SummaryPipelineWorkflow with id "pipeline-" + episodeId; returns {episodeId}), GET
    /episodes (list from getAllEpisodes, sorted newest-first), GET /episodes/{id} (one row),
    GET /episodes/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- TranscriptTasks.java declaring three Task<R> constants:
    EXTRACT_QUOTES = Task.name("Extract quotes").description("Pull representative quotes from
      the transcript using scanTranscript and fetchSpeakerTurn").resultConformsTo(QuoteSet.class);
    SEGMENT_TOPICS = Task.name("Segment topics").description("Cluster the extracted quotes
      into topic groups using clusterQuotes and labelCluster").resultConformsTo(Segmentation.class);
    WRITE_SUMMARY = Task.name("Write summary").description("Draft summary sections for each
      topic cluster using draftSection and writeOverallTakeaway").resultConformsTo(EpisodeSummary.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {EXTRACT, SEGMENT, SUMMARIZE}. Each function-tool method is annotated
  with the constant phase, e.g. @FunctionTool(name = "scanTranscript", phase = Phase.EXTRACT)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- ExtractTools.java — @FunctionTool scanTranscript(String transcriptText) -> List<Quote>
  reading from src/main/resources/sample-data/transcripts/*.json keyed by episodeTitle;
  @FunctionTool fetchSpeakerTurn(String turnId) -> Quote reading from the matching
  transcript entry's speaker turns.

- SegmentTools.java — @FunctionTool clusterQuotes(List<Quote>) -> List<TopicCluster>
  (deterministic clustering — assign each Quote to a cluster keyed by the first significant
  noun phrase; 2-5 clusters typical); @FunctionTool labelCluster(List<Quote>) -> String
  (returns a 3-5-word label for the cluster).

- SummarizeTools.java — @FunctionTool draftSection(TopicCluster, List<Quote>) -> SummarySection
  (heading from cluster topicLabel, body composed from the quote texts joined with linking
  phrases, supportingQuoteIds from the cluster's quoteIds); @FunctionTool
  writeOverallTakeaway(List<SummarySection>) -> String (1-sentence overarching insight).

- PhaseGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.phase attribute, looks up the EpisodeEntity status by episodeId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("phase-violation: <tool> requires <precondition>, saw
  <status>"). On reject ALSO calls EpisodeEntity.recordGuardrailRejection(phase, tool, reason)
  so the rejection is visible in the UI's rejection-log strip and in the audit log.

- FidelityScorer.java — pure deterministic logic (no LLM). Inputs: EpisodeSummary,
  Segmentation, QuoteSet. Outputs: EvalResult with score and rationale. Four checks, one
  point per check satisfied, starting from a base of 1: cluster coverage (every TopicCluster
  has a SummarySection), quote attribution (every SummarySection has >= 1 supportingQuoteId),
  quote provenance (every supportingQuoteId appears in QuoteSet.quotes[].quoteId), and
  section parity (sections.size() == clusters.size()). Score range 1-5. Rationale names the
  largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9821 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/episodes.jsonl with 5 seeded episode lines covering the
  three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/transcripts/*.json — three files keyed by seeded episode,
  each carrying 8-12 Quote entries with deterministic content so ExtractTools.scanTranscript
  returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root with data.data_classes.pii = false (quotes are
  editorial content, not personal data; speakers are public podcast guests), decisions.authority_level
  = recommend-only (the summary is advisory), oversight.human_in_loop = true (an editor reads
  the summary before publishing), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "fabricated-quote", "missing-cluster-coverage",
  "phase-violation", "quote-thin-section"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/transcript-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Podcast Transcript Summarizer",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of episode cards; right = selected-episode detail with title header, quotes table,
  topic clusters with quote assignments, summary sections, eval-score chip, rejection-log
  strip). Browser title exactly: <title>Akka Sample: Podcast Transcript Summarizer</title>.
  No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(episodeId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    extract-quotes.json — 6 QuoteSet entries, each with 6-10 Quote items per seeded episode.
      Each entry's tool_calls array contains 2-4 calls: 1 scanTranscript(transcript) + 1-3
      fetchSpeakerTurn(turnId) calls. Plus 1 deliberately PHASE-VIOLATING entry whose
      tool_calls array starts with a draftSection(...) call (a SUMMARIZE-phase tool called
      during the EXTRACT phase) — the guardrail rejects it, the mock then falls through to a
      normal extract sequence. The mock should select the violating entry on the FIRST
      iteration of every 3rd episode (modulo seed) so J2 is reproducible.
    segment-topics.json — 6 Segmentation entries paired one-to-one with the extract entries,
      each with 2-4 TopicCluster items, with tool_calls containing clusterQuotes +
      labelCluster in order.
    write-summary.json — 6 EpisodeSummary entries paired one-to-one. Each carries 2-4
      SummarySection items matching the paired Segmentation clusters, supportingQuoteIds
      referencing the paired QuoteSet quoteIds, tool_calls containing draftSection (one per
      cluster) + writeOverallTakeaway. Plus 1 deliberately FABRICATED-QUOTE entry whose first
      SummarySection cites a quoteId absent from the paired QuoteSet — the workflow's evalStep
      scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(episodeId) helper makes per-episode selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TranscriptAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TranscriptTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (extractStep
  60s, segmentStep 60s, summarizeStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the EpisodeRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: TranscriptTasks.java with EXTRACT_QUOTES, SEGMENT_TOPICS, WRITE_SUMMARY constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9821 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (TranscriptAgent). The
  on-decision eval is rule-based (FidelityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-tool-call guardrail (PhaseGuardrail) is the runtime mechanism that enforces the
  phase order. Do NOT conditionally register tools per task — the guardrail is the gate.
- Task dependency is carried by typed task results: extractStep writes QuoteSet onto the
  entity, segmentStep reads it and builds the SEGMENT task's instruction context from it,
  summarizeStep reads both. The agent itself is stateless across phases.
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
