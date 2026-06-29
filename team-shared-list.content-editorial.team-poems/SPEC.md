# SPEC — team-poems-multi-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Team Poems Multi-Agent.
**One-line pitch:** Submit a writing prompt; a poetry director breaks it into stanza assignments on a shared board, and several poet agents claim stanzas on their own, draft lines, pass a quality gate, and coordinate with each other when they need to match tone or meter.

## 2. What this blueprint demonstrates

The **team-shared-list** coordination pattern wired with Akka's first-party primitives: a direction workflow runs a poetry-director agent that writes a list of stanza assignments onto a shared board, and a fixed roster of poet agents each runs its own durable loop that pulls open assignments off that board, claims them atomically, produces a drafted verse, and marks them done. No central editor hands work out — the poets self-organise around the shared board, and the board entity's single-writer guarantee is what stops two poets claiming the same stanza. The blueprint also demonstrates two lightweight governance mechanisms appropriate for a creative multi-agent toy: a **content guardrail** that refuses outputs violating configurable content rules, and an **editorial pause** that lets the operator freeze the whole team.

## 3. User-facing flows

The user opens the App UI tab and submits a writing prompt via the form (a title plus a free-text inspiration note).

1. The system logs the submission on `PromptQueue` and a `Consumer` starts a `DirectionWorkflow` for the poem.
2. The `PoetryDirector` agent decomposes the prompt into a `VersePlan` — a list of stanza assignments, each with a title, a verse brief, the form (haiku / free-verse / rhyming-couplet), and the titles of assignments it depends on for continuity. The workflow writes one `StanzaEntity` per assignment (status `OPEN`) and moves the poem to `ASSIGNED`.
3. The poet roster (`poet-1`, `poet-2`, `poet-3`) is already running — each poet is a `PoetWorkflow` that polls the shared `VerseBoardView` for an `OPEN` assignment whose dependencies are all `DONE`.
4. A poet claims an eligible assignment. The claim is an atomic command on that assignment's `StanzaEntity`; if two poets race for the same stanza, one wins and the other goes back to polling.
5. The winning poet runs the `PoetAgent`, which writes a `DraftVerse` (the stanza lines plus a note on the approach). Every draft passes through a content guardrail; an output that violates the content rules is refused and the stanza is recorded `BLOCKED`.
6. The draft runs through a quality gate. On pass, the stanza becomes `DONE`. On fail, the poet retries up to a fixed limit, then the stanza becomes `BLOCKED`.
7. If the poet reports needing another poet's output for rhythm or continuity, it raises a `CoordinationRequest`. The stanza goes `BLOCKED`, a `CoordinationMessage` lands in the target poet's `CoordinationMailbox`, and the poet who can answer posts a reply that unblocks the waiting stanza.
8. When every stanza on a poem is `DONE`, the poem moves to `PUBLISHED`.

A `PromptSimulator` (TimedAction) drips a sample writing prompt every 60 seconds so the App UI is not empty when first loaded. A `StuckStanzaMonitor` (TimedAction) releases any stanza that has been claimed but idle for more than two minutes, returning it to `OPEN` so another poet can pick it up.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PoetryDirector` | `AutonomousAgent` | Decomposes a writing prompt into a list of stanza assignments with ordering constraints. | `DirectionWorkflow` | returns typed `VersePlan` to workflow |
| `PoetAgent` | `AutonomousAgent` | Claims and drafts one stanza assignment; returns a `DraftVerse`; may raise a `CoordinationRequest`. Run as several instances (`poet-1`, `poet-2`, `poet-3`). | `PoetWorkflow` | returns typed result to workflow |
| `DirectionWorkflow` | `Workflow` | Runs `PoetryDirector`, creates one `StanzaEntity` per assignment, moves the poem to `ASSIGNED`. | `PromptSubmissionConsumer` | `PoetryDirector`, `StanzaEntity`, `PoemEntity` |
| `PoetWorkflow` | `Workflow` | Per-poet durable loop: poll → claim → draft → quality gate → done/blocked/coordination. | `Bootstrap` (one instance per poet id) | `StanzaEntity`, `PoetAgent`, `CoordinationMailbox`, `EditorialControl` |
| `PoemEntity` | `EventSourcedEntity` | Poem lifecycle: prompt, stanza ids, status. | `DirectionWorkflow`, `PoemCompletionMonitor` | `VerseBoardView` is stanza-side; poem read via endpoint |
| `StanzaEntity` | `EventSourcedEntity` | One per stanza. Atomic `claim`; full stanza lifecycle. | `DirectionWorkflow`, `PoetWorkflow`, `StuckStanzaMonitor` | `VerseBoardView` |
| `CoordinationMailbox` | `EventSourcedEntity` | One per poet. Inbound coordination messages and replies. | `PoetWorkflow` | `PoetryEndpoint` |
| `PromptQueue` | `EventSourcedEntity` | Single instance; logs each submitted prompt for replay/audit. | `PoetryEndpoint`, `PromptSimulator` | `PromptSubmissionConsumer` |
| `EditorialControl` | `KeyValueEntity` | Operator pause flag for the whole team. | `PoetryEndpoint` | `PoetWorkflow`, `PoetAgent` guardrail |
| `VerseBoardView` | `View` | The shared assignment list read model the poets poll and the UI streams. | `StanzaEntity` events | `PoetWorkflow`, `PoetryEndpoint` |
| `PromptSubmissionConsumer` | `Consumer` | Listens to `PromptQueue` events; starts a `DirectionWorkflow` per submission. | `PromptQueue` events | `DirectionWorkflow` |
| `PromptSimulator` | `TimedAction` | Drips a sample writing prompt every 60 s. | scheduler | `PromptQueue` |
| `StuckStanzaMonitor` | `TimedAction` | Every 30 s, releases stanzas claimed but idle > 2 min back to `OPEN`. | scheduler | `StanzaEntity` |
| `PoetryEndpoint` | `HttpEndpoint` | `/api/*` — submit, list stanzas, get poem, coordination reply, pause/resume, SSE. | — | `PromptQueue`, `VerseBoardView`, `PoemEntity`, `CoordinationMailbox`, `EditorialControl` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts one `PoetWorkflow` per poet id. | — | `PoetWorkflow`, scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record WritingPrompt(String promptId, String title, String inspiration, String submittedBy) {}

record StanzaSpec(String title, String verseBrief, VerseForm form, List<String> dependsOn) {}
record VersePlan(String planNote, List<StanzaSpec> stanzas) {}

record DraftLine(int lineNumber, String text) {}
record CoordinationRequest(String toPoet, String question) {}
record DraftVerse(String stanzaId, List<DraftLine> lines, String approach,
                 Optional<CoordinationRequest> coordinationRequest) {}

record QualityReport(boolean passed, List<String> issues, Instant ranAt) {}
record CoordinationMessage(String messageId, String fromPoet, String toPoet,
                           String stanzaId, String question, Instant sentAt,
                           Optional<String> reply, Optional<Instant> repliedAt) {}
```

### Entity state — `Stanza` (state of `StanzaEntity`, basis of the view row)

```java
record Stanza(
    String stanzaId,
    String poemId,
    String title,
    String verseBrief,
    VerseForm form,
    List<String> dependsOn,
    StanzaStatus status,
    Optional<String>  claimedBy,
    Optional<Instant> claimedAt,
    Optional<String>  approachNote,
    Optional<Integer> lineCount,
    Optional<QualityReport> qualityReport,
    Optional<String>  blockedReason,
    Optional<Instant> completedAt,
    Instant createdAt
) {}

enum StanzaStatus { OPEN, CLAIMED, DRAFTING, IN_REVIEW, DONE, BLOCKED }
enum VerseForm    { HAIKU, FREE_VERSE, RHYMING_COUPLET, SONNET_QUATRAIN }
```

### Entity state — `Poem` (state of `PoemEntity`)

```java
record Poem(
    String poemId,
    String title,
    String inspiration,
    String submittedBy,
    PoemStatus status,
    List<String> stanzaIds,
    Optional<String> planNote,
    Instant createdAt,
    Optional<Instant> publishedAt
) {}

enum PoemStatus { RECEIVED, ASSIGNED, DRAFTING, PUBLISHED }
```

### Events

`StanzaEntity`: `StanzaCreated`, `StanzaClaimed`, `DraftingStarted`, `VerseSubmitted`, `QualityPassed`, `QualityFailed`, `StanzaBlocked`, `StanzaReleased`, `StanzaCompleted`.
`PoemEntity`: `PoemCreated`, `PoemAssigned`, `PoemDrafting`, `PoemPublished`.
`CoordinationMailbox`: `CoordinationPosted`, `CoordinationReplied`.
`PromptQueue`: `PromptSubmitted`.
`EditorialControl` is a `KeyValueEntity` — state transitions are command-driven, no events.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/poems` — body `{ title, inspiration, submittedBy? }` → `{ poemId }`. Logs the submission and starts direction.
- `GET /api/poems/{id}` — one poem with its stanza ids and status.
- `GET /api/stanzas` — list all stanzas. Optional `?poemId=...` and `?status=...`, filtered client-side.
- `GET /api/stanzas/sse` — server-sent events stream of every stanza change.
- `GET /api/mailbox/{poetId}` — coordination messages for a poet.
- `POST /api/mailbox/{poetId}/messages/{messageId}/reply` — body `{ reply }`. Posts a reply that unblocks the waiting stanza.
- `POST /api/control/pause` — body `{ reason, by }`. Sets the operator pause flag.
- `POST /api/control/resume` — clears the pause flag.
- `GET /api/control` — current pause state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Team Poems Multi-Agent</title>`.

- **Overview** — eyebrow "Overview" + headline "Team Poems Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, stanza state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill. (This baseline carries no controls; the table renders empty with a placeholder row.)
- **App UI** — a writing prompt submission form, a live board grouped by status column (Open / Claimed / Drafting / In review / Done / Blocked), per-stanza cards showing the claiming poet and approach note, a coordination-mailbox panel with reply boxes, and a Pause / Resume control.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This is a toy creative sample with no industry-bound regulatory anchors. The generated system wires:

- (No controls — `controls: []` in `eval-matrix.yaml`.) The content guardrail and the operator pause are present as lightweight application logic but are not registered as formal governance controls in this baseline. A deployer adding this system to a regulated context should add controls via the eval matrix.

## 9. Agent prompts

- `PoetryDirector` → `prompts/poetry-director.md`. Decomposes a writing prompt into a dependency-ordered list of stanza assignments.
- `PoetAgent` → `prompts/poet.md`. Claims a stanza assignment, drafts the verse lines, raises a coordination request when it needs another poet's output for tone or meter continuity.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a writing prompt; the poetry director assigns stanzas; poets claim and complete them; the poem reaches `PUBLISHED`. The board updates live via SSE.
2. **J2** — Two poets poll at the same instant; each stanza ends up claimed by exactly one poet (no double-claim).
3. **J3** — A poet raises a coordination request; the stanza goes `BLOCKED`, a message lands in the peer's mailbox, and a reply unblocks it.
4. **J4** — The editor pauses the team; claiming pauses; resume continues the drafting.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named team-poems-multi-agent demonstrating the
team-shared-list × content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact team-shared-list-content-editorial-team-poems.
Java package io.akka.samples.teampoemsmultiagent. Akka 3.6.0. HTTP port 9299.

Components to wire (exactly):
- 2 AutonomousAgents:
  * PoetryDirector — definition() with capability(PoemTasks.of(DIRECT)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/poetry-director.md.
    Returns VersePlan{planNote, stanzas: List<StanzaSpec{title, verseBrief,
    form: VerseForm, dependsOn: List<String>}>}.
  * PoetAgent — definition() with capability(PoemTasks.of(DRAFT)
    .maxIterationsPerTask(3)). System prompt from prompts/poet.md. Returns
    DraftVerse{stanzaId, lines: List<DraftLine{lineNumber, text}>, approach,
    coordinationRequest: Optional<CoordinationRequest{toPoet, question}>}. Run as
    several runtime instances addressed by instanceId (poet-1, poet-2, poet-3) —
    ONE agent class, several instance ids; never generate three agent classes.
    Register a before-tool-call guardrail on this agent (content guardrail — refuses
    outputs that invoke tool calls for external publishing while the team is paused).

- 1 Workflow DirectionWorkflow with steps directStep -> createStanzasStep ->
  assignedStep. directStep calls forAutonomousAgent(PoetryDirector.class, promptId)
  .runSingleTask(DIRECT.instructions(prompt)) then forTask(taskId)
  .result(DIRECT). createStanzasStep writes one StanzaEntity per StanzaSpec with a
  deterministic stanzaId = poemId + "-s" + index and calls
  PoemEntity.recordPlan(stanzaIds, planNote). assignedStep ends the workflow.
  Override settings() with stepTimeout(directStep, 90s).

- 1 Workflow PoetWorkflow, one durable instance per poet id. State carries
  poetId and the currently-claimed stanzaId (Optional). Steps:
  pollStep -> claimStep -> draftStep -> reviewStep -> finishStep, with a
  self-scheduled resume timer for idling. pollStep: read EditorialControl; if
  paused, schedule a 5s resume timer and pause. Otherwise query
  VerseBoardView.getAllStanzas, pick the oldest OPEN stanza whose dependsOn titles
  are all DONE; if none, schedule a 5s resume timer and pause; else go to
  claimStep. claimStep: call StanzaEntity.claim(poetId); if the entity reports the
  stanza is no longer OPEN (lost the race), go back to pollStep; else
  StanzaEntity.startDrafting and go to draftStep. draftStep (stepTimeout 90s): call
  forAutonomousAgent(PoetAgent.class, poetId).runSingleTask(
  DRAFT.instructions(stanza)). If a guardrail refusal comes back, call
  StanzaEntity.block(reason) and go to pollStep. On a CoordinationRequest in the
  result, call CoordinationMailbox(toPoet).post(message) and
  StanzaEntity.block("waiting on poet: " + toPoet), then go to pollStep.
  Otherwise call StanzaEntity.submitVerse(draft) and go to reviewStep.
  reviewStep: run the deterministic QualityChecker over the draft; on pass call
  StanzaEntity.passQuality(report) (-> DONE) and go to finishStep; on fail up to 2
  retries call StanzaEntity.failQuality(report) and loop draftStep; after retries
  exhausted call StanzaEntity.block("quality checks failed") and go to pollStep.
  finishStep: clear the claimed stanza and go to pollStep.

- 4 EventSourcedEntities:
  * StanzaEntity holding Stanza state. Commands: createStanza, claim,
    startDrafting, submitVerse, passQuality, failQuality, block, release, getStanza.
    claim emits StanzaClaimed ONLY when status==OPEN; otherwise it is a no-op that
    returns the current Stanza so the caller can detect the lost race. StanzaStatus
    enum: OPEN, CLAIMED, DRAFTING, IN_REVIEW, DONE, BLOCKED. VerseForm enum:
    HAIKU, FREE_VERSE, RHYMING_COUPLET, SONNET_QUATRAIN. Events: StanzaCreated,
    StanzaClaimed, DraftingStarted, VerseSubmitted, QualityPassed, QualityFailed,
    StanzaBlocked, StanzaReleased, StanzaCompleted. emptyState() returns
    Stanza.initial("", "") with NO commandContext() reference.
  * PoemEntity holding Poem state. Commands: createPoem, recordPlan,
    startDrafting, publish, getPoem. PoemStatus enum: RECEIVED, ASSIGNED,
    DRAFTING, PUBLISHED. Events: PoemCreated, PoemAssigned, PoemDrafting,
    PoemPublished. emptyState() returns Poem.initial("") with NO
    commandContext() reference.
  * CoordinationMailbox, id = poetId. Commands: post(CoordinationMessage),
    reply(messageId, text), getMailbox. Events: CoordinationPosted,
    CoordinationReplied. State is a List<CoordinationMessage>.
  * PromptQueue, single instance "default". Command submitPrompt(WritingPrompt)
    emitting PromptSubmitted{promptId, title, inspiration, submittedBy,
    submittedAt}.

- 1 KeyValueEntity EditorialControl, single instance "default", holding
  {paused: boolean, pausedReason: Optional<String>, pausedBy: Optional<String>,
  pausedAt: Optional<Instant>}. Commands: pause(reason, by), resume, getControl.

- 1 View VerseBoardView with row type StanzaRow (mirrors Stanza minus the heavy
  DraftLine contents — keep approachNote and lineCount only). Table updater
  consumes StanzaEntity events. ONE query getAllStanzas SELECT * AS stanzas FROM
  verse_board. No WHERE status filter (Akka cannot auto-index enum columns —
  Lesson 2); callers filter client-side.

- 1 Consumer PromptSubmissionConsumer subscribed to PromptQueue events; on
  PromptSubmitted calls PoemEntity.createPoem then starts a
  DirectionWorkflow with the poemId as the workflow id.

- 2 TimedActions:
  * PromptSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/writing-prompts.jsonl and calls
    PromptQueue.submitPrompt. Stops cycling when the file is exhausted (wraps).
  * StuckStanzaMonitor — every 30s, queries VerseBoardView.getAllStanzas, finds
    stanzas in CLAIMED or DRAFTING whose claimedAt is older than 2 minutes,
    and calls StanzaEntity.release (-> OPEN, clears claimedBy).

- 2 HttpEndpoints:
  * PoetryEndpoint at /api with POST /poems, GET /poems/{id},
    GET /stanzas (client-side filter by poemId/status), GET /stanzas/sse,
    GET /mailbox/{poetId},
    POST /mailbox/{poetId}/messages/{id}/reply,
    POST /control/pause, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule PromptSimulator and StuckStanzaMonitor;
  start one PoetWorkflow instance per poet id (poet-1, poet-2, poet-3).

Companion files:
- PoemTasks.java declaring two Task<R> constants: DIRECT (resultConformsTo
  VersePlan) and DRAFT (resultConformsTo DraftVerse).
- Domain records WritingPrompt, StanzaSpec, VersePlan, DraftLine,
  CoordinationRequest, DraftVerse, QualityReport, CoordinationMessage in domain/.
- QualityChecker.java (deterministic, in application/) — a pure function over a
  DraftVerse returning a QualityReport: passes when every DraftLine has a
  non-blank text, the approach is non-empty, and no line contains a literal
  "PLACEHOLDER" or "TODO"; otherwise fails with one issue string per offending
  line.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9299 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also a poet-roster setting
  team-poems.poets = ["poet-1","poet-2","poet-3"] read by Bootstrap.
- src/main/resources/sample-events/writing-prompts.jsonl with 6 canned writing
  prompts (e.g., a haiku sequence on seasons, a free-verse ode to a river, a
  rhyming quatrain about a city at night, a sonnet on patience, a prose poem on
  memory, a collaborative verse on the passage of time).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with zero controls and a matching empty
  simplified_view list.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/poetry-director.md and prompts/poet.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Team Poems Multi-Agent",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs matching the formal exemplar:
  Overview, Architecture (4 mermaid diagrams + click-to-expand component table
  with syntax-highlighted Java snippets), Risk Survey (7 sections from
  governance.html with answers from risk-survey.yaml; unanswered .qb opacity
  0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows and a colored mechanism pill in the ID column; empty
  placeholder row when no controls are present), App UI (prompt form + a board
  with one column per StanzaStatus, a coordination-mailbox panel with reply boxes,
  and a Pause/Resume control). Browser title exactly:
  <title>Akka Sample: Team Poems Multi-Agent</title>. No subtitle on the Overview
  tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory; gone
        when the session ends.
- NEVER write the key value to any file. Akka records only the REFERENCE.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with a per-agent dispatch on agent class name.
  Each branch reads a JSON file from src/main/resources/mock-responses/:
    poetry-director.json — 4-6 VersePlan entries, each with a planNote and
      3-5 StanzaSpec items with believable stanza titles (e.g., "Opening image",
      "Rising tension", "Turn", "Resolution"), VerseForm values, and dependsOn
      references that exercise the dependency gate.
    poet.json — 4-6 DraftVerse entries. Most have 2-5 DraftLine items with
      non-empty, PLACEHOLDER-free text and a one-sentence approach note and an
      empty coordinationRequest. Include 1-2 entries whose coordinationRequest
      is present (toPoet "poet-2", a concrete rhythm question) to exercise J3,
      and include 1 entry with a blank DraftLine text so the quality check fails
      deterministically for J-quality-gate coverage.
- MockModelProvider.seedFor(stanzaId) makes selection deterministic per stanza id.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion PoemTasks Task<R> constants (Lesson 7).
- View has no WHERE filter on the enum status column; filter client-side
  (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9299 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- UI is a single self-contained static-resources/index.html — no ui/ folder,
  no package.json, no npm build (Lesson 17).
- No forbidden words in user-facing text (Lessons 21, 22, 23).
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
