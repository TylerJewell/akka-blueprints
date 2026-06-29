# SPEC — book-writer-fanout

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–12 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Book Writer.
**One-line pitch:** The user types a book topic; an outline agent plans the chapters, a fresh writing agent drafts each chapter, and a final step joins the chapters into one manuscript shown live in the UI.

## 2. What this blueprint demonstrates

The composite-multi-team coordination pattern in the content-editorial domain: a planning agent produces a chapter list, then for each chapter the workflow spawns a fresh writing pass with its own session and per-chapter state, and a consolidation step merges the results into a single document. The governance pattern wires three controls — a before-tool-call guardrail on the web-search tool, a per-chapter quality eval before consolidation, and a documentation completeness gate before a book is marked done.

## 3. User-facing flows

1. The user opens the App UI tab and types a book topic, then submits. A book appears in `REQUESTED` state with its id.
2. The outline agent runs; the book moves to `OUTLINED` and shows a title plus a numbered chapter list, each chapter `PENDING`.
3. The workflow drafts chapters one at a time; each chapter moves `DRAFTING` → `DRAFTED`, and a quality score appears beside it.
4. Once every chapter is drafted, the consolidation step joins them; the book moves to `COMPLETED` and the full manuscript renders.
5. Without any interaction, the simulator drips a sample topic on a timer, so books keep arriving.

These map to `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| OutlineAgent | AutonomousAgent | Turn a topic into a titled chapter plan | BookWritingWorkflow | BookEntity |
| ChapterAgent | AutonomousAgent | Draft one chapter from its brief; call the guarded web-search tool | BookWritingWorkflow | ChapterEntity, WebSearchEndpoint |
| BookWritingWorkflow | Workflow | Orchestrate outline → per-chapter writing → consolidate | BookRequestConsumer | OutlineAgent, ChapterAgent, BookEntity, ChapterEntity |
| BookEntity | EventSourcedEntity | Book lifecycle and manuscript | BookWritingWorkflow | BooksView |
| ChapterEntity | EventSourcedEntity | Per-chapter draft + quality score | BookWritingWorkflow, ChapterEvalConsumer | BooksView |
| BooksView | View | Read model the UI lists and streams | BookEntity, ChapterEntity | BookEndpoint |
| BookRequestConsumer | Consumer | Start a workflow per submitted topic | BookRequestQueue | BookWritingWorkflow |
| BookRequestQueue | EventSourcedEntity | Record each incoming topic submission | BookEndpoint, RequestSimulator | BookRequestConsumer |
| ChapterEvalConsumer | Consumer | Score each chapter when drafted (eval-event) | ChapterEntity | ChapterEntity |
| RequestSimulator | TimedAction | Drip a sample topic on a timer | — | BookRequestQueue |
| WebSearchEndpoint | HttpEndpoint | In-process web-search facade with a before-tool-call guard | ChapterAgent | — |
| BookEndpoint | HttpEndpoint | `/api` surface — submit, list, single, SSE, metadata | UI, clients | BookRequestQueue, BooksView |
| AppEndpoint | HttpEndpoint | Serve `/` redirect and `/app/*` static UI | browser | — |

Names are used verbatim by `/akka:specify`.

## 5. Data model

See `reference/data-model.md` for the full field list. Summary:

- `Book` (BookEntity state + view row): `id`, `Optional<String> topic`, `BookStatus status`, `Optional<String> title`, `List<ChapterSummary> chapters`, `Optional<Integer> chapterCount`, `Optional<Integer> chaptersDrafted`, `Optional<String> manuscript`, plus `Optional<Instant>` for `outlinedAt`, `consolidatedAt`, `completedAt`, `failedAt`, and `Optional<String> failureReason`.
- `Chapter` (ChapterEntity state): `chapterId`, `bookId`, `int index`, `String title`, `String brief`, `ChapterStatus status`, `Optional<String> draftMarkdown`, `Optional<Double> qualityScore`, `Optional<Instant> draftedAt`, `Optional<Instant> evaluatedAt`.
- `ChapterSummary` (embedded in the view row): `chapterId`, `int index`, `String title`, `String status`, `Optional<Double> qualityScore`.
- `BookStatus` enum: `REQUESTED · OUTLINED · WRITING · CONSOLIDATING · COMPLETED · FAILED`.
- `ChapterStatus` enum: `PENDING · DRAFTING · DRAFTED · APPROVED · REJECTED`.
- Book events: `BookRequested`, `OutlineRecorded`, `ChapterProgressed`, `ManuscriptConsolidated`, `BookCompleted`, `BookFailed`.
- Chapter events: `ChapterDrafted`, `ChapterEvaluated`, `ChapterRejected`.

Every nullable lifecycle field is `Optional<T>` (Lesson 6).

## 6. API contract

Full payloads in `reference/api-contract.md`. Top-level surface:

```
POST /api/books                  -> { bookId }
GET  /api/books ?status=...      -> { books: [Book, ...] }
GET  /api/books/{bookId}         -> Book
GET  /api/books/sse              -> Server-Sent Events of Book
POST /api/search                 -> { results: [...] }   (in-process, guarded)

GET  /api/metadata/eval-matrix   -> text/yaml
GET  /api/metadata/risk-survey   -> text/yaml
GET  /api/metadata/readme        -> text/markdown

GET  /                           -> 302 /app/index.html
GET  /app/{*path}                -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are fine. Browser title: `<title>Akka Sample: Book Writer</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). Mermaid diagrams on the Architecture tab carry the state-label colour overrides and edge-label `overflow:visible` fixes (Lesson 24). The App UI tab submits a topic, lists books over SSE, and expands a book to show its chapter list, per-chapter quality scores, and the consolidated manuscript. See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1 — guardrail · before-tool-call:** the WebSearchEndpoint checks every query from ChapterAgent against the book topic and an injection denylist before returning results; off-topic or injection-shaped queries are refused.
- **E1 — eval-event · on-decision-eval:** ChapterEvalConsumer fires on `ChapterDrafted`, scores the chapter, and records the score on ChapterEntity; consolidation reads the scores.
- **A1 — ci-gate · documentation-gate:** the manuscript completeness check — every planned chapter present and non-empty — gates the `COMPLETED` transition, and an integration test asserts the same before deploy.

## 9. Agent prompts

- `prompts/outline-agent.md` — system prompt for OutlineAgent: produce a titled, numbered chapter plan from a topic.
- `prompts/chapter-agent.md` — system prompt for ChapterAgent: draft one chapter from its brief, using the guarded web-search tool only for on-topic lookups.

## 10. Acceptance

Inline (full set in `reference/user-journeys.md`):

1. **J1 — Submit and outline.** POST `/api/books` with a topic; within ~30 s the book is `OUTLINED` with a title and a non-empty chapter list.
2. **J2 — Chapters draft.** Each chapter transitions to `DRAFTED` with a non-empty draft and a quality score.
3. **J3 — Consolidate.** When all chapters are drafted, the book reaches `COMPLETED` with a manuscript that contains every chapter title.
4. **J4 — Guard blocks off-topic search.** A ChapterAgent search query unrelated to the topic is refused by the before-tool-call guard and does not return results.

---

## 11. Implementation directives

```
Create a sample named book-writer-fanout demonstrating the
composite-multi-team x content-editorial cell. Runs out of the box (no external
services; the web-search service is modeled in-process). Maven group
io.akka.samples. Maven artifact book-writer-fanout. Java package
io.akka.samples.bookwriterfanout. Akka 3.6.0. HTTP port 9537.

Components to wire (exactly):
- 2 AutonomousAgents:
  - OutlineAgent: definition() with capability(TaskAcceptance.of(OUTLINE)
    .maxIterationsPerTask(3)). Returns a typed BookOutline{title,
    List<ChapterPlan> chapters} where ChapterPlan{title, brief}.
  - ChapterAgent: definition() with capability(TaskAcceptance.of(WRITE_CHAPTER)
    .maxIterationsPerTask(4)) and a web_search tool that POSTs to the in-process
    WebSearchEndpoint. Returns a typed ChapterDraft{markdown}.
- 1 Workflow BookWritingWorkflow with steps outlineStep -> writeChaptersStep
  -> consolidateStep. outlineStep calls forAutonomousAgent(OutlineAgent.class,
  "outline-"+bookId).runSingleTask(OUTLINE) then forTask(taskId).result(OUTLINE);
  records the outline on BookEntity and creates one ChapterEntity per chapter
  (id "chapter-"+bookId+"-"+index). writeChaptersStep iterates chapters in order;
  for each it calls forAutonomousAgent(ChapterAgent.class,
  "chapter-"+bookId+"-"+index).runSingleTask(WRITE_CHAPTER) with a fresh session,
  records the draft on the ChapterEntity, and advances BookEntity.chaptersDrafted.
  consolidateStep reads every ChapterEntity, asserts each is DRAFTED/APPROVED with
  non-empty markdown (documentation gate), joins them into one manuscript, records
  ManuscriptConsolidated then BookCompleted on BookEntity; on a failed gate records
  BookFailed. Override settings() with stepTimeout(120s) on outlineStep,
  writeChaptersStep, and consolidateStep, and defaultStepRecovery(maxRetries(2)
  .failoverTo(BookWritingWorkflow::error)).
- 1 EventSourcedEntity BookEntity holding a Book record (fields in
  reference/data-model.md; all lifecycle fields Optional<T>). Events: BookRequested,
  OutlineRecorded, ChapterProgressed, ManuscriptConsolidated, BookCompleted,
  BookFailed. Commands: requestBook, recordOutline, progressChapter,
  recordConsolidation, complete, fail, getBook. emptyState() returns
  Book.initial("") with no commandContext() reference.
- 1 EventSourcedEntity ChapterEntity holding a Chapter record. Events:
  ChapterDrafted, ChapterEvaluated, ChapterRejected. Commands: createChapter,
  recordDraft, recordEvaluation, reject, getChapter.
- 1 EventSourcedEntity BookRequestQueue with command enqueueRequest(topic)
  emitting BookRequestQueued.
- 1 View BooksView with row type Book, table updater consuming BookEntity AND
  ChapterEntity events (chapter events update the matching ChapterSummary inside
  the book row). ONE query getAllBooks: SELECT * AS books FROM books_view. No
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side.
  Provide streamAllBooks for SSE.
- 1 Consumer BookRequestConsumer subscribed to BookRequestQueue events; starts a
  BookWritingWorkflow with a fresh UUID per event.
- 1 Consumer ChapterEvalConsumer subscribed to ChapterEntity ChapterDrafted
  events; computes a quality score (length, topic-term coverage, structure) and
  calls ChapterEntity.recordEvaluation; if below threshold 0.5 calls reject.
- 1 TimedAction RequestSimulator (every 45s, reads next line from
  src/main/resources/sample-events/book-topics.jsonl and calls
  BookRequestQueue.enqueueRequest).
- 3 HttpEndpoints:
  - WebSearchEndpoint at /api/search: accepts {bookId, query}; loads the book
    topic via BooksView; before returning canned results from
    src/main/resources/sample-docs/*.md it runs the before-tool-call guard —
    rejects (403) when the query shares no salient term with the topic or matches
    an injection denylist ("ignore previous", "system prompt", URL-exfiltration
    patterns). This realises control G1.
  - BookEndpoint at /api with books (POST submit, GET list filtered client-side,
    GET single, GET sse) and the three /api/metadata/* endpoints serving files
    from src/main/resources/metadata/.
  - AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- BookWritingTasks.java declaring two Task<R> constants: OUTLINE
  (resultConformsTo BookOutline), WRITE_CHAPTER (resultConformsTo ChapterDraft).
- BookOutline(String title, List<ChapterPlan> chapters),
  ChapterPlan(String title, String brief), ChapterDraft(String markdown).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9537
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/book-topics.jsonl with 6 canned topic lines.
- src/main/resources/sample-docs/*.md with 4 canned search-result documents.
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md}
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1, E1, A1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): elevator pitch, component
  inventory, integration descriptor, how to run, the tabs, API contract, license.
  NO governance-mechanisms section. NO configuration section.
- src/main/resources/static-resources/index.html — single self-contained HTML
  file (no ui/, no npm). Five tabs (Overview, Architecture, Risk Survey, Eval
  Matrix, App UI). Match the governance.html visual style (dark / yellow accent /
  Instrument Sans / dot-grid). Mermaid Lesson-24 CSS overrides and attribute-based
  tab switching from Lesson 26.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for ANTHROPIC_API_KEY
  / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
      outputs (OutlineAgent -> BookOutline with 3-5 ChapterPlan entries;
      ChapterAgent -> ChapterDraft with 3-6 paragraphs of markdown; see
      src/main/resources/mock-responses/{outline-agent,chapter-agent}.json with
      4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
      /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in the Claude session; passed
      to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider per-agent shapes (option a):
- OutlineAgent -> BookOutline{title:"<topic> — A Field Guide", chapters:[3-5 x
  ChapterPlan{title, brief}]}.
- ChapterAgent -> ChapterDraft{markdown:"## <chapter title>\n\n<3-6 paragraphs>"}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches verbatim.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout(120s).
- Lesson 6: Optional<T> for every nullable lifecycle field on a view row record.
- Lesson 7: each AutonomousAgent has its Task<R> constants in BookWritingTasks.
- Lesson 8: verify model names current before locking them in application.conf.
- Lesson 9: run command is "/akka:build" (slash command), never "mvn akka:run".
- Lesson 10: explicit http-port 9537 in application.conf.
- Lesson 11: never render the corpus source platform in any user-facing surface.
- Lesson 12: UI fits the 1080px content column; no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box"; never T1-T4.
- Lesson 23: no competitor brand names anywhere in user-facing text.
- Lesson 24: mermaid state-label colour overrides + edge-label overflow:visible
  + transitionLabelColor #cccccc.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never by
  NodeList index; delete removed tabs, never display:none zombie panels.
- emptyState() never calls commandContext().
- WorkflowSettings is nested inside Workflow — no import needed.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and the PLAN.md.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9537`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — a missing API key (offer the five key-sourcing options from Section 11; the mock-model path needs none), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
