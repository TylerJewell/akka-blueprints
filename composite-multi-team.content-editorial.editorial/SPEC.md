# SPEC — editorial-desk

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Editorial Desk.
**One-line pitch:** Submit a story topic; an editor-in-chief agent assigns a brief, a research desk delegates to several researchers and synthesises their notes, a writing desk's writers claim sections off a shared board, a review panel scores the draft, and a passing verdict assembles and publishes the final article.

## 2. What this blueprint demonstrates

The **composite-multi-team** coordination pattern wired with Akka's first-party primitives: a top-level editor-in-chief pipeline delegates three stages — research, writing, review — to three section desks, and each desk runs a *different* internal coordination capability over one shared document workspace. The research desk uses **delegation**: a research lead plans subtopics and fans out one researcher per subtopic, then synthesises their notes. The writing desk uses a **team** over a shared list: a writing lead plans sections onto a board, and a roster of writer loops claim open sections atomically and fill them. The review desk uses **moderation**: a panel of reviewer instances each score the draft on one axis, and a deterministic rule turns the panel's notes into a pass-or-revise verdict. The blueprint also demonstrates four governance mechanisms an editorial pipeline needs: a **before-agent-response guardrail** on the final article, a **before-tool-call guardrail** on every write into the shared document workspace, a non-blocking **stage eval** on each stage result, and a non-blocking **post-publication compliance review** of published articles.

## 3. User-facing flows

The user opens the App UI tab and submits a story via the form (a topic plus an optional requester name).

1. The system logs the submission on `StoryQueue` and a `Consumer` starts an `EditorialWorkflow` for the story.
2. The `EditorInChief` agent assigns the story — it produces an `EditorialBrief` (an angle, key questions, and the target sections). The document moves to `ASSIGNED`.
3. **Research desk (delegation).** The `ResearchLead` plans a `ResearchPlan` (a list of subtopics). The workflow runs one `Researcher` instance per subtopic; each researcher writes a `ResearchNote` into the shared document workspace through `DocumentTools`. The lead then synthesises the notes into a `ResearchDigest`. The document moves to `RESEARCHED`.
4. **Writing desk (team over a shared list).** The `WritingLead` plans a `SectionPlan`. The workflow writes one `SectionEntity` per section onto the board (status `OPEN`) and the document moves to `WRITING`. The writer roster (`writer-1`, `writer-2`) is already running — each writer is a `WriterWorkflow` that polls the shared `SectionBoardView` for an `OPEN` section, claims it atomically, runs the `Writer` agent to produce the section content through `DocumentTools`, and marks the section `WRITTEN`. If two writers race for the same section, one wins and the other returns to polling.
5. When every section for the story is `WRITTEN`, the workflow assembles a `Draft` and the document moves to `DRAFTED`.
6. **Review desk (moderation).** The workflow runs the review panel (`factcheck`, `style`, `legal`) — one `Reviewer` instance per axis, each writing a `ReviewNote` (a per-axis `PASS`/`REVISE` verdict and comments). A deterministic `ModerationRule` aggregates the notes into a `ReviewVerdict`: `PASS` only when every axis passes, otherwise `REVISE` with the list of sections to redo. The document moves to `REVIEWING` then to `APPROVED` or back to `WRITING` for one revision round.
7. **Publish.** On a passing verdict the workflow runs `EditorInChief` once more to assemble the final `Article` from the approved draft. A before-agent-response guardrail vets the article before it is persisted. The document moves to `PUBLISHED` with a generated URL.
8. A `StageEvalConsumer` fires a non-blocking quality eval on each stage result (research digest, draft, review verdict) and records a `StageEval` on the document, surfaced beside the story in the UI.
9. After an article is `PUBLISHED`, a compliance officer can post a post-publication `ComplianceReview` (cleared or flagged, with comments). It is recorded against the published article and shown in the UI; it does not change the published state — the review is on the loop, not in it.

A `StorySimulator` (TimedAction) drips a sample topic every 60 seconds so the App UI is not empty when first loaded. A `StuckSectionMonitor` (TimedAction) releases any section that has been claimed but idle for more than two minutes, returning it to `OPEN` so another writer can pick it up.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EditorInChief` | `AutonomousAgent` | Assigns the story (produces `EditorialBrief`) and, at publish, assembles the final `Article` under an output guardrail. | `EditorialWorkflow` | returns typed result to workflow; writes via `DocumentEntity` |
| `ResearchLead` | `AutonomousAgent` | Plans research subtopics and synthesises the researchers' notes into a `ResearchDigest`. | `EditorialWorkflow` | returns `ResearchPlan` / `ResearchDigest` |
| `Researcher` | `AutonomousAgent` | Produces one `ResearchNote` for one subtopic; writes it into the workspace via `DocumentTools`. Run as several instances (one per subtopic). | `EditorialWorkflow` | `DocumentTools` → `DocumentEntity` |
| `WritingLead` | `AutonomousAgent` | Plans the article sections into a `SectionPlan`. | `EditorialWorkflow` | returns `SectionPlan` |
| `Writer` | `AutonomousAgent` | Writes one section's content via `DocumentTools`. Run as several instances (`writer-1`, `writer-2`). | `WriterWorkflow` | `DocumentTools` → `SectionEntity` |
| `Reviewer` | `AutonomousAgent` | Scores the draft on one axis; returns a `ReviewNote`; writes it via `DocumentTools`. Run as several instances (`factcheck`, `style`, `legal`). | `EditorialWorkflow` | `DocumentTools` → `DocumentEntity` |
| `EditorialWorkflow` | `Workflow` | The editor-in-chief pipeline: brief → research → writing → review → publish, with a revision loop. | `StoryRequestConsumer` | all agents, `DocumentEntity`, `SectionEntity`, `SectionBoardView` |
| `WriterWorkflow` | `Workflow` | Per-writer durable loop: poll the board → claim a section atomically → write → mark written. | `Bootstrap` (one instance per writer id) | `SectionEntity`, `Writer`, `SectionBoardView` |
| `DocumentEntity` | `EventSourcedEntity` | The shared document workspace: one story's lifecycle, brief, digest, draft, verdict, article, stage evals, compliance review. | `EditorialWorkflow`, `DocumentTools`, `StageEvalConsumer`, `EditorialEndpoint` | `DocumentBoardView` |
| `SectionEntity` | `EventSourcedEntity` | One per writing section. Atomic `claim`; section lifecycle. | `EditorialWorkflow`, `WriterWorkflow`, `DocumentTools`, `StuckSectionMonitor` | `SectionBoardView` |
| `StoryQueue` | `EventSourcedEntity` | Single instance; logs each submitted topic for replay/audit. | `EditorialEndpoint`, `StorySimulator` | `StoryRequestConsumer` |
| `DocumentBoardView` | `View` | The list-of-stories read model the UI streams. | `DocumentEntity` events | `EditorialEndpoint` |
| `SectionBoardView` | `View` | The shared section board the writers poll and the UI shows. | `SectionEntity` events | `WriterWorkflow`, `EditorialWorkflow`, `EditorialEndpoint` |
| `StoryRequestConsumer` | `Consumer` | Listens to `StoryQueue` events; starts an `EditorialWorkflow` per submission. | `StoryQueue` events | `EditorialWorkflow`, `DocumentEntity` |
| `StageEvalConsumer` | `Consumer` | Listens to `DocumentEntity` stage events; runs a non-blocking eval and records a `StageEval`. | `DocumentEntity` events | `DocumentEntity` |
| `StorySimulator` | `TimedAction` | Drips a sample story topic every 60 s. | scheduler | `StoryQueue` |
| `StuckSectionMonitor` | `TimedAction` | Every 30 s, releases sections claimed but idle > 2 min back to `OPEN`. | scheduler | `SectionEntity` |
| `EditorialEndpoint` | `HttpEndpoint` | `/api/*` — submit, get story, list stories, list sections, compliance review, SSE, metadata. | — | `StoryQueue`, `DocumentBoardView`, `SectionBoardView`, `DocumentEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts one `WriterWorkflow` per writer id. | — | `WriterWorkflow`, scheduler |

Companion classes that are not Akka components: `EditorialTasks` (the `Task<R>` constants), `DocumentTools` (the function tool the worker agents call to write into the workspace; the before-tool-call guardrail vets it), and `ModerationRule` (the deterministic pure function that aggregates the review panel's notes into a verdict).

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record StoryBrief(String storyId, String topic, String requestedBy) {}

record EditorialBrief(String angle, List<String> keyQuestions, List<String> targetSections) {}

record ResearchPlan(List<String> subtopics) {}
record ResearchNote(String subtopic, String findings, List<String> sources) {}
record ResearchDigest(String summary, List<String> keyFacts) {}

record SectionSpec(String title, String brief) {}
record SectionPlan(String approach, List<SectionSpec> sections) {}
record SectionDraft(String sectionId, String title, String content, int wordCount) {}
record Draft(String headline, List<SectionDraft> sections, int wordCount) {}

record ReviewNote(String axis, ReviewOutcome outcome, String comments) {}
record ReviewVerdict(ReviewOutcome outcome, List<ReviewNote> notes, List<String> mustRevise) {}

record Article(String headline, String body, String byline, Instant assembledAt) {}

record StageEval(String stage, int score, List<String> flags, Instant evaluatedAt) {}
record ComplianceReview(String reviewedBy, ComplianceOutcome outcome, String comments, Instant reviewedAt) {}
```

### Entity state — `Document` (state of `DocumentEntity`, basis of the board row)

```java
record Document(
    String storyId,
    String topic,
    String requestedBy,
    StoryStatus status,
    Optional<EditorialBrief>   brief,
    Optional<ResearchDigest>   researchDigest,
    List<String>               sectionIds,
    Optional<Draft>            draft,
    Optional<ReviewVerdict>    reviewVerdict,
    Optional<Article>          article,
    Optional<String>           publishedUrl,
    Optional<Instant>          publishedAt,
    List<StageEval>            stageEvals,
    Optional<ComplianceReview> complianceReview,
    int                        revisionCount,
    Instant                    createdAt
) {}

enum StoryStatus { SUBMITTED, ASSIGNED, RESEARCHING, RESEARCHED, WRITING, DRAFTED, REVIEWING, APPROVED, PUBLISHED }
```

### Entity state — `Section` (state of `SectionEntity`, basis of the section-board row)

```java
record Section(
    String sectionId,
    String storyId,
    String title,
    String brief,
    SectionStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  content,
    Optional<Integer> wordCount,
    Optional<Instant> writtenAt,
    Instant createdAt
) {}

enum SectionStatus { OPEN, CLAIMED, WRITTEN }
enum ReviewOutcome { PASS, REVISE }
enum ComplianceOutcome { CLEARED, FLAGGED }
```

### Events

`DocumentEntity`: `DocumentCreated`, `BriefAssigned`, `ResearchSynthesized`, `SectionsPlanned`, `DraftAssembled`, `ReviewCompleted`, `RevisionRequested`, `ArticlePublished`, `StageEvaluated`, `ComplianceReviewRecorded`.
`SectionEntity`: `SectionCreated`, `SectionClaimed`, `SectionWritten`, `SectionReleased`.
`StoryQueue`: `StorySubmitted`.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/stories` — body `{ topic, requestedBy? }` → `202 { storyId }`. Logs the submission and starts the pipeline.
- `GET /api/stories` — list all stories. Optional `?status=...`, filtered client-side.
- `GET /api/stories/{id}` — one story with its brief, digest, draft, verdict, article, stage evals, and compliance review.
- `GET /api/stories/sse` — server-sent events stream of every document change.
- `GET /api/stories/{id}/sections` — the sections on the board for one story.
- `GET /api/sections/sse` — server-sent events stream of every section change.
- `POST /api/stories/{id}/compliance-review` — body `{ reviewedBy, outcome, comments }`. Records a post-publication compliance review; allowed only when the story is `PUBLISHED`. Non-blocking.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Editorial Desk</title>`.

- **Overview** — eyebrow "Overview" + headline "Editorial Desk"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, document state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a story submission form; a live story board grouped by status with per-story cards showing the brief angle, the research digest, the section progress, the review verdict, the stage-eval scores, and (when published) the article and a compliance-review box; plus a section-board panel grouped by section status (Open / Claimed / Written) showing the claiming writer.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`llm-content-evaluation` flavor, on `EditorInChief`): when the editor-in-chief assembles the final `Article` at publish, a guardrail vets the response before it is persisted — it checks minimum length, refuses profanity and banned topics, and requires the article to carry a byline. A failing article blocks the publish step and the document stays `APPROVED` with the guardrail reason recorded. Blocking.
- **G2 — before-tool-call guardrail** (`tool-permission` flavor, on the worker agents that call `DocumentTools`): every write into the shared document workspace — a research note, a section's content, a review note — passes through a guardrail that refuses a write addressed to a story or section the agent was not assigned, a write to a story that is already `PUBLISHED`, or an oversized payload. A refused write records the stage with the guardrail reason and the agent returns without a partial write. Blocking.
- **E1 — stage eval** (`eval-event`, `on-decision-eval` flavor): `StageEvalConsumer` fires on each stage result event (`ResearchSynthesized`, `DraftAssembled`, `ReviewCompleted`) and runs a deterministic quality eval — a score and a list of flags — recorded as a `StageEval` on the document. Informational; surfaces context beside the story. Non-blocking.
- **HO1 — post-publication compliance review** (`hotl`, `live-compliance-review` flavor): after an article publishes, a compliance officer can post a `ComplianceReview` through `POST /api/stories/{id}/compliance-review`. It is recorded against the published article and shown in the UI but does not change the published state — the reviewer is on the loop, not gating it. Non-blocking, system-level.

## 9. Agent prompts

- `EditorInChief` → `prompts/editor-in-chief.md`. Assigns the story brief and assembles the final article.
- `ResearchLead` → `prompts/research-lead.md`. Plans research subtopics and synthesises the notes.
- `Researcher` → `prompts/researcher.md`. Produces one research note for one subtopic.
- `WritingLead` → `prompts/writing-lead.md`. Plans the article sections.
- `Writer` → `prompts/writer.md`. Writes one section's content.
- `Reviewer` → `prompts/reviewer.md`. Scores the draft on one axis.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a topic; the editor-in-chief assigns a brief; the research desk delegates and synthesises; the writing desk fills sections on a shared board; the review panel scores the draft; a passing verdict assembles and publishes the article. The board updates live via SSE.
2. **J2** — Two writers poll at the same instant; each section ends up claimed by exactly one writer (no double-claim).
3. **J3** — The review panel returns a `REVISE` verdict; the story loops back to `WRITING` for one revision round, then publishes on the second pass.
4. **J4** — A worker agent attempts a refused document-workspace write; the before-tool-call guardrail refuses it before execution and the stage is recorded with the guardrail reason.
5. **J5** — After an article publishes, a compliance officer posts a compliance review; it is recorded against the published article and shown in the UI without changing the published state.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named editorial-desk demonstrating the
composite-multi-team × content-editorial cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact editorial-desk. Java
package io.akka.samples.editorialdesk. Akka 3.6.0. HTTP port 9934.

Components to wire (exactly):
- 6 AutonomousAgents (each extends akka.javasdk.agent.autonomous.AutonomousAgent;
  never silently downgrade to Agent):
  * EditorInChief — definition() with capability(TaskAcceptance.of(ASSIGN)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(FINALIZE)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/editor-in-chief.md.
    ASSIGN returns EditorialBrief{angle, keyQuestions, targetSections}. FINALIZE
    returns Article{headline, body, byline, assembledAt}. Register a
    before-agent-response guardrail on this agent that validates the FINALIZE
    output (control G1).
  * ResearchLead — capability(TaskAcceptance.of(PLAN_RESEARCH)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(SYNTHESIZE)
    .maxIterationsPerTask(2)). System prompt from prompts/research-lead.md.
    PLAN_RESEARCH returns ResearchPlan{subtopics}. SYNTHESIZE returns
    ResearchDigest{summary, keyFacts}.
  * Researcher — capability(TaskAcceptance.of(RESEARCH).maxIterationsPerTask(2)).
    System prompt from prompts/researcher.md. Returns ResearchNote{subtopic,
    findings, sources}. Run as several runtime instances addressed by instanceId
    (storyId + "-r" + subtopicIndex) — ONE agent class, several instance ids.
    Equipped with the DocumentTools function tool; the G2 before-tool-call
    guardrail is registered on this agent.
  * WritingLead — capability(TaskAcceptance.of(PLAN_SECTIONS)
    .maxIterationsPerTask(2)). System prompt from prompts/writing-lead.md.
    Returns SectionPlan{approach, sections: List<SectionSpec{title, brief}>}.
  * Writer — capability(TaskAcceptance.of(WRITE_SECTION).maxIterationsPerTask(3)).
    System prompt from prompts/writer.md. Returns SectionDraft{sectionId, title,
    content, wordCount}. Run as several runtime instances (writer-1, writer-2) —
    ONE agent class, several instance ids; never generate two writer classes.
    Equipped with DocumentTools; the G2 before-tool-call guardrail is registered
    on this agent.
  * Reviewer — capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)).
    System prompt from prompts/reviewer.md. Returns ReviewNote{axis, outcome,
    comments}. Run as several runtime instances (factcheck, style, legal) — ONE
    agent class, several instance ids; the axis is passed as the instruction.
    Equipped with DocumentTools; the G2 before-tool-call guardrail is registered
    on this agent.

- 1 Workflow EditorialWorkflow, one instance per storyId. State carries storyId,
  topic, brief (Optional), digest (Optional), sectionIds (List), draft
  (Optional), verdict (Optional), revisionCount, and stage. Steps:
  briefStep -> researchStep -> writingStep -> reviewStep -> publishStep, with a
  self-scheduled resume timer for the writing wait.
  * briefStep: call forAutonomousAgent(EditorInChief.class, storyId)
    .runSingleTask(ASSIGN.instructions(topic)) then forTask(taskId).result(ASSIGN);
    call DocumentEntity.assignBrief(brief). Go to researchStep.
  * researchStep (delegation): call ResearchLead PLAN_RESEARCH -> ResearchPlan.
    For each subtopic, call forAutonomousAgent(Researcher.class, storyId + "-r" +
    i).runSingleTask(RESEARCH.instructions(subtopic)) then result(RESEARCH); each
    researcher writes its ResearchNote into DocumentEntity via DocumentTools. Then
    call ResearchLead SYNTHESIZE over the notes -> ResearchDigest; call
    DocumentEntity.recordDigest(digest). Go to writingStep.
  * writingStep (team over a shared list): call WritingLead PLAN_SECTIONS ->
    SectionPlan. Write one SectionEntity per SectionSpec with a deterministic
    sectionId = storyId + "-s" + index (status OPEN) and call
    DocumentEntity.recordSections(sectionIds). Then WAIT: query
    SectionBoardView.getAllSections, filter to this story; if every section is
    WRITTEN, assemble a Draft from the section contents and call
    DocumentEntity.assembleDraft(draft) and go to reviewStep; otherwise schedule a
    5s resume timer and pause. The writer roster fills the sections independently
    (see WriterWorkflow).
  * reviewStep (moderation): for each axis in [factcheck, style, legal] call
    forAutonomousAgent(Reviewer.class, storyId + "-" + axis).runSingleTask(
    REVIEW.instructions(axis, draft)) then result(REVIEW); each reviewer writes
    its ReviewNote via DocumentTools. Pass the notes through ModerationRule ->
    ReviewVerdict. Call DocumentEntity.recordReview(verdict). On PASS go to
    publishStep. On REVISE, if revisionCount < 1, call
    DocumentEntity.requestRevision(mustRevise), reset the named sections to OPEN on
    their SectionEntity, increment revisionCount, and go back to writingStep; if
    revisionCount already 1, accept the draft as-is and go to publishStep
    (one bounded revision round).
  * publishStep: call EditorInChief FINALIZE -> Article (guarded by G1). If the
    guardrail refuses, call DocumentEntity.recordPublishBlock(reason) and end the
    workflow with the document left APPROVED. Otherwise call
    DocumentEntity.publish(article, url) with url =
    "https://desk.example/" + storyId. End the workflow.
  Override settings() with stepTimeout(briefStep, 60s),
  stepTimeout(researchStep, 120s), stepTimeout(reviewStep, 90s),
  stepTimeout(publishStep, 60s). writingStep is a poll step with a short timeout
  and a resume timer.

- 1 Workflow WriterWorkflow, one durable instance per writer id (writer-1,
  writer-2). State carries writerId and the currently-claimed sectionId
  (Optional). Steps: pollStep -> claimStep -> writeStep -> finishStep, with a
  self-scheduled resume timer for idling.
  * pollStep: query SectionBoardView.getAllSections; pick the oldest OPEN section;
    if none, schedule a 5s resume timer and pause; else go to claimStep.
  * claimStep: call SectionEntity.claim(writerId); if the entity reports the
    section is no longer OPEN (lost the race), go back to pollStep; else go to
    writeStep.
  * writeStep (stepTimeout 90s): call forAutonomousAgent(Writer.class, writerId)
    .runSingleTask(WRITE_SECTION.instructions(section)); the writer writes the
    content via DocumentTools and the workflow calls
    SectionEntity.recordWritten(content, wordCount). If a guardrail refusal comes
    back, call SectionEntity.release (back to OPEN) and go to pollStep. Go to
    finishStep.
  * finishStep: clear the claimed section and go to pollStep.

- 3 EventSourcedEntities:
  * DocumentEntity holding Document state, one per storyId. Commands:
    createDocument, assignBrief, recordDigest, recordSections, assembleDraft,
    recordReview, requestRevision, publish, recordPublishBlock, recordStageEval,
    recordComplianceReview, appendResearchNote, appendReviewNote, getDocument.
    StoryStatus enum: SUBMITTED, ASSIGNED, RESEARCHING, RESEARCHED, WRITING,
    DRAFTED, REVIEWING, APPROVED, PUBLISHED. Events: DocumentCreated,
    BriefAssigned, ResearchSynthesized, SectionsPlanned, DraftAssembled,
    ReviewCompleted, RevisionRequested, ArticlePublished, StageEvaluated,
    ComplianceReviewRecorded. emptyState() returns Document.initial("") with NO
    commandContext() reference. recordComplianceReview is accepted only when
    status==PUBLISHED.
  * SectionEntity holding Section state, one per sectionId. Commands:
    createSection, claim, recordWritten, release, getSection. claim emits
    SectionClaimed ONLY when status==OPEN; otherwise it is a no-op that returns the
    current Section so the caller can detect the lost race. SectionStatus enum:
    OPEN, CLAIMED, WRITTEN. Events: SectionCreated, SectionClaimed, SectionWritten,
    SectionReleased. emptyState() returns Section.initial("", "") with NO
    commandContext() reference.
  * StoryQueue, single instance "default". Command submitStory(StoryBrief)
    emitting StorySubmitted{storyId, topic, requestedBy, submittedAt}.

- 2 Views:
  * DocumentBoardView with row type DocumentRow (mirrors Document but drops the
    heavy Article body and ResearchNote contents — keeps brief angle, digest
    summary, sectionIds, draft headline + wordCount, verdict outcome, stageEvals,
    publishedUrl, complianceReview). Table updater consumes DocumentEntity events.
    ONE query getAllDocuments SELECT * AS documents FROM document_board. No WHERE
    status filter (Akka cannot auto-index enum columns — Lesson 2); callers filter
    client-side. Add a streamAllDocuments query for the SSE endpoint.
  * SectionBoardView with row type SectionRow (mirrors Section minus the heavy
    content — keeps wordCount only, plus a contentPresent boolean). Table updater
    consumes SectionEntity events. ONE query getAllSections SELECT * AS sections
    FROM section_board. No WHERE filter; callers filter client-side. Add a
    streamAllSections query for the SSE endpoint.

- 2 Consumers:
  * StoryRequestConsumer subscribed to StoryQueue events; on StorySubmitted calls
    DocumentEntity.createDocument then starts an EditorialWorkflow with the storyId
    as the workflow id.
  * StageEvalConsumer subscribed to DocumentEntity events; on ResearchSynthesized,
    DraftAssembled, and ReviewCompleted it runs a deterministic StageEvaluator
    (pure function, NOT an LLM call) producing a StageEval{stage, score, flags,
    evaluatedAt} and calls DocumentEntity.recordStageEval(eval). Ignore all other
    DocumentEntity events. This is control E1.

- 2 TimedActions:
  * StorySimulator — every 60s, reads the next line from
    src/main/resources/sample-events/story-topics.jsonl (wraps when exhausted) and
    calls StoryQueue.submitStory.
  * StuckSectionMonitor — every 30s, queries SectionBoardView.getAllSections,
    finds sections in CLAIMED whose claimedAt is older than 2 minutes, and calls
    SectionEntity.release (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * EditorialEndpoint at /api with POST /stories, GET /stories (client-side filter
    by status), GET /stories/{id}, GET /stories/sse, GET /stories/{id}/sections,
    GET /sections/sse, POST /stories/{id}/compliance-review (accepted only when the
    story is PUBLISHED; returns 409 otherwise), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

- 1 service-setup Bootstrap: schedule StorySimulator and StuckSectionMonitor;
  start one WriterWorkflow instance per writer id (writer-1, writer-2).

Companion files:
- EditorialTasks.java declaring the Task<R> constants: ASSIGN (resultConformsTo
  EditorialBrief), FINALIZE (Article), PLAN_RESEARCH (ResearchPlan), RESEARCH
  (ResearchNote), SYNTHESIZE (ResearchDigest), PLAN_SECTIONS (SectionPlan),
  WRITE_SECTION (SectionDraft), REVIEW (ReviewNote).
- DocumentTools.java — the function tool the worker agents (Researcher, Writer,
  Reviewer) call to write into the shared workspace: appendResearchNote(storyId,
  note), writeSection(sectionId, content), appendReviewNote(storyId, note). Each
  method routes through the DocumentEntity / SectionEntity. The G2 before-tool-call
  guardrail vets every call: refuse when the storyId/sectionId is not the one the
  agent was assigned, when the target story is already PUBLISHED, or when the
  payload exceeds a size cap.
- ModerationRule.java (deterministic, in application/) — a pure function over a
  List<ReviewNote> returning a ReviewVerdict: outcome is PASS only when every note
  is PASS; otherwise REVISE with mustRevise = the distinct titles named in the
  REVISE notes' comments (or all sections when none are named). This is the
  moderation aggregation; it is NOT an LLM call.
- StageEvaluator.java (deterministic, in application/) — a pure function used by
  StageEvalConsumer: scores a stage result 0-100 and lists flags (e.g., digest
  with fewer than three keyFacts, draft under a word-count floor, a review with
  any REVISE axis). NOT an LLM call.
- Domain records StoryBrief, EditorialBrief, ResearchPlan, ResearchNote,
  ResearchDigest, SectionSpec, SectionPlan, SectionDraft, Draft, ReviewNote,
  ReviewVerdict, Article, StageEval, ComplianceReview, and the enums StoryStatus,
  SectionStatus, ReviewOutcome, ComplianceOutcome in domain/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9934
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also
  editorial-desk.writers = ["writer-1","writer-2"] and editorial-desk.review-axes
  = ["factcheck","style","legal"] read by Bootstrap and EditorialWorkflow.
- src/main/resources/sample-events/story-topics.jsonl with 6 canned story topics
  (each a self-contained explainer — e.g., "How tides work", "The history of the
  shipping container", "Why bread rises", "What a compiler does", "How vaccines
  are tested", "The economics of a coffee shop").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (G1 before-agent-
  response guardrail, G2 before-tool-call guardrail, E1 eval-event on-decision-
  eval, HO1 hotl live-compliance-review) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/editor-in-chief.md, prompts/research-lead.md, prompts/researcher.md,
  prompts/writing-lead.md, prompts/writer.md, prompts/reviewer.md loaded at agent
  startup as system prompts.
- README.md at the project root: title "Akka Sample: Editorial Desk", one-line
  pitch, prerequisites (host software: none), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs matching the formal exemplar:
  Overview, Architecture (4 mermaid diagrams + click-to-expand component table
  with syntax-highlighted Java snippets), Risk Survey (7 sections from
  governance.html with answers from risk-survey.yaml; unanswered .qb opacity
  0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows and a colored mechanism pill in the ID column), App UI
  (story form + a story board grouped by StoryStatus showing brief/digest/section
  progress/verdict/stage-eval scores/article/compliance box, and a section-board
  panel grouped by SectionStatus showing the claiming writer). Browser title
  exactly: <title>Akka Sample: Editorial Desk</title>. No subtitle on the Overview
  tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider block
        below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session
        ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka records
  only the REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing the
  ModelProvider interface with a per-agent dispatch on the agent class name (or the
  Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json, picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return
  shape. A MockModelProvider.seedFor(id) helper makes the selection deterministic
  per story/section id so the same input produces the same output across restarts.
- Per-agent mock-response shapes for THIS blueprint:
    editor-in-chief.json — 4-6 entries with two variants. ASSIGN variants are
      EditorialBrief with an angle, 3-4 keyQuestions, and 3-5 targetSections.
      FINALIZE variants are Article with a headline, a multi-paragraph body, and a
      non-empty byline. Include 1 FINALIZE entry whose body is too short so the G1
      output guardrail blocks it for J-guardrail coverage on the publish path.
    research-lead.json — 4-6 entries. PLAN_RESEARCH variants are ResearchPlan with
      3-4 subtopics. SYNTHESIZE variants are ResearchDigest with a one-paragraph
      summary and 4-6 keyFacts.
    researcher.json — 4-6 ResearchNote entries, each a subtopic, a short findings
      paragraph, and 1-3 plausible sources. Include 1 entry whose findings target a
      mismatched storyId to exercise the G2 before-tool-call guardrail refusal.
    writing-lead.json — 4-6 SectionPlan entries, each an approach sentence and 3-5
      SectionSpec items (title + one-sentence brief) for an explainer article.
    writer.json — 4-6 SectionDraft entries with a title, 1-3 short paragraphs of
      non-empty content, and a wordCount. Include 1 entry whose content is empty to
      exercise a release-and-retry.
    reviewer.json — 6-8 ReviewNote entries spanning the three axes. Most are PASS;
      include at least one REVISE note per the J-revision journey, naming a section
      title in its comments so ModerationRule can populate mustRevise.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion EditorialTasks Task<R> constants (Lesson 7).
- Views have no WHERE filter on the enum status column; filter client-side
  (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9934 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24 (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc). Without these,
  state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel
  attribute, NEVER by NodeList index (Lesson 26). No "hidden" zombie panels in the
  DOM — delete removed tabs, do not display:none them.
- UI is a single self-contained static-resources/index.html — no ui/ folder, no
  package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export
  block. Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing tone,
  competitor brand names (Lessons 21, 22, 23).
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars plus the mock-LLM option from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
