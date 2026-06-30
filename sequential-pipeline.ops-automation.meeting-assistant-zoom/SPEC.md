# SPEC — meeting-assistant-zoom

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Zoom AI Meeting Assistant.
**One-line pitch:** A user submits a Zoom meeting transcript; one `MeetingAgent` walks it through three task phases — **TRANSCRIBE** raw audio into a redacted transcript, **SUMMARIZE** into a structured summary with action items, **DISPATCH** follow-up tasks and calendar events — with each phase gated on the prior phase's recorded output and participant PII stripped before any text reaches the LLM.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an ops-automation domain. One `MeetingAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the TRANSCRIBE task's typed output becomes the SUMMARIZE task's instruction context; the SUMMARIZE task's typed output becomes the DISPATCH task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`pii-sanitizer`** runs before the TRANSCRIBE task. It scans the raw transcript for participant names, email addresses, and phone numbers, replaces each occurrence with a stable pseudonym (`[PERSON-1]`, `[PERSON-2]`, …), and records a `RedactionAudit` alongside the sanitized text. The LLM in all three phases sees only the sanitized transcript; the original is stored encrypted on `MeetingEntity` but is never forwarded to any LLM call.
- A **`before-tool-call` guardrail** (`ScopeGuardrail`) sits between the agent and every tool call. It enforces two orthogonal constraints: (1) phase order — a DISPATCH-phase tool called while the entity has not yet recorded `SummaryProduced` is rejected before the tool body runs; (2) write scope — any calendar or task write whose invitee or assignee is not a member of the recorded participant list is rejected with a `scope-violation` error, preventing the agent from creating follow-ups for people who were not in the meeting.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs enforce the dependency contract while the two governance mechanisms protect the data leaving the pipeline.

## 3. User-facing flows

The user opens the App UI tab.

1. The user uploads a Zoom transcript file (or picks one of three seeded meetings — `Q2 Engineering Retrospective`, `Sales Pipeline Review`, `Product Roadmap Kickoff`).
2. The user clicks **Process meeting**. The UI POSTs to `/api/meetings` and receives a `meetingId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `TRANSCRIBING` — the PII sanitizer has run and the workflow has handed the TRANSCRIBE task to the agent.
4. Within ~15–25 s the card reaches `SUMMARIZING` — the typed `Transcript` with `redactionAudit` is visible in the card detail. The workflow recorded `TranscriptProduced` and started the SUMMARIZE task.
5. Within ~15–25 s more the card reaches `DISPATCHING`. The `MeetingSummary` is visible (summary paragraph, agenda items, action items with tentative assignees).
6. Within ~15–25 s more the card reaches `EVALUATED`. The right pane now shows the full `MeetingPackage` — summary, action items with owners and due dates, scheduled follow-up events — plus an eval score chip (1–5) and a one-line rationale.
7. The user can submit another meeting; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MeetingEndpoint` | `HttpEndpoint` | `/api/meetings/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `MeetingEntity`, `MeetingView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `MeetingEntity` | `EventSourcedEntity` | Per-meeting lifecycle: created → transcribing → transcribed → summarizing → summarized → dispatching → dispatched → evaluated. Source of truth. | `MeetingEndpoint`, `MeetingPipelineWorkflow` | `MeetingView` |
| `MeetingPipelineWorkflow` | `Workflow` | One workflow per meeting. Steps: `transcribeStep` → `summarizeStep` → `dispatchStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `MeetingEndpoint` after `CREATED` | `MeetingAgent`, `MeetingEntity` |
| `MeetingAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `MeetingTasks.java`: `TRANSCRIBE_MEETING` → `Transcript`, `SUMMARIZE_MEETING` → `MeetingSummary`, `DISPATCH_FOLLOWUPS` → `MeetingPackage`. Each task is registered with the phase-appropriate function tools. | invoked by `MeetingPipelineWorkflow` | returns typed results |
| `TranscribeTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `segmentTranscript(rawText)` and `labelSpeakers(segments)`. Reads from `src/main/resources/sample-data/transcripts/*.json` for deterministic offline output. | called from TRANSCRIBE task | returns `List<Segment>` |
| `SummarizeTools` | function-tools class | Implements `extractActionItems(transcript)` and `identifyDecisions(transcript)`. Pure in-memory transformations. | called from SUMMARIZE task | returns `List<ActionItem>` / `List<Decision>` |
| `DispatchTools` | function-tools class | Implements `scheduleFollowUp(attendees, title, dueDate)` and `createTask(assignee, description, dueDate)`. | called from DISPATCH task | returns `FollowUpEvent` / `TaskRecord` |
| `ScopeGuardrail` | `before-tool-call` guardrail (registered on `MeetingAgent`) | Enforces phase order and write scope. Reads the in-flight task's declared phase and the current `MeetingEntity` status; also checks that any write target (attendee, assignee) appears in the recorded participant list. | every tool call on every task | accept / structured-reject |
| `PiiSanitizer` | plain class (no Akka primitive) | Runs before `transcribeStep` begins. Scans raw transcript text for PII patterns (names via NER heuristic, email, phone), replaces each with a stable pseudonym, and returns `SanitizedTranscript{text, redactionAudit}`. | called by `MeetingPipelineWorkflow.transcribeStep` before `runSingleTask` | returns sanitized text + audit |
| `ActionItemScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `MeetingPackage`, `MeetingSummary`, `Transcript`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `MeetingView` | `View` | Read model: one row per meeting for the UI. | `MeetingEntity` events | `MeetingEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Segment(String speaker, String text, Instant startTime, Instant endTime) {}

record RedactionEntry(String original, String pseudonym, String patternType) {}

record RedactionAudit(int totalRedactions, List<RedactionEntry> entries) {}

record Transcript(
    List<Segment> segments,
    RedactionAudit redactionAudit,
    Instant capturedAt
) {}

record ActionItem(
    String actionId,
    String description,
    String assigneePseudonym,
    Optional<LocalDate> dueDate
) {}

record Decision(String decisionId, String text, Instant decidedAt) {}

record MeetingSummary(
    String summaryText,
    List<ActionItem> actionItems,
    List<Decision> decisions,
    Instant summarizedAt
) {}

record FollowUpEvent(
    String eventId,
    String title,
    List<String> attendeePseudonyms,
    LocalDate scheduledDate
) {}

record TaskRecord(
    String taskId,
    String assigneePseudonym,
    String description,
    LocalDate dueDate
) {}

record MeetingPackage(
    String title,
    String summaryText,
    List<ActionItem> actionItems,
    List<FollowUpEvent> followUpEvents,
    List<TaskRecord> tasks,
    Instant packagedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record MeetingRecord(
    String meetingId,
    Optional<String> meetingTitle,
    Optional<Transcript> transcript,
    Optional<MeetingSummary> summary,
    Optional<MeetingPackage> meetingPackage,
    Optional<EvalResult> eval,
    MeetingStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MeetingStatus {
    CREATED, TRANSCRIBING, TRANSCRIBED, SUMMARIZING, SUMMARIZED,
    DISPATCHING, DISPATCHED, EVALUATED, FAILED
}
```

Events on `MeetingEntity`: `MeetingCreated`, `TranscribeStarted`, `TranscriptProduced`, `SummarizeStarted`, `SummaryProduced`, `DispatchStarted`, `FollowUpsDispatched`, `EvaluationScored`, `ScopeRejected`, `MeetingFailed`.

Every nullable lifecycle field on the `MeetingRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/meetings` — body `{ meetingTitle, rawTranscript }` → `{ meetingId }`.
- `GET /api/meetings` — list all meetings, newest-first.
- `GET /api/meetings/{id}` — one meeting.
- `GET /api/meetings/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Zoom AI Meeting Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of meetings (status pill + title + age) and a right pane with the selected meeting's detail — title, redaction audit, transcript segments, summary with action items, meeting package, eval score chip, and a guardrail-rejection log strip if any scope rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — `pii-sanitizer`**: `PiiSanitizer` runs synchronously inside `MeetingPipelineWorkflow.transcribeStep`, before `runSingleTask` is called. It scans the raw `rawTranscript` string using a three-pass heuristic: (1) email-address regex `[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}`; (2) phone-number regex `\+?[\d\s\-().]{7,15}`; (3) a simple NLP heuristic for capitalized two-word sequences that appear across multiple segments (likely participant names). Each match is assigned a stable pseudonym keyed by a SHA-1 of the original text — `[PERSON-1]`, `[EMAIL-1]`, `[PHONE-1]`, etc. — and the mapping is stored in `RedactionAudit`. The sanitized text is what the `TaskDef.instructions` carry into the LLM for all three phases. The original text is stored on the entity under `rawTranscriptEncrypted` (AES-GCM, key from the service secret) but is never forwarded to the LLM. The redaction count is shown in the App UI's redaction-audit badge on the transcript panel.
- **G1 — `before-tool-call` guardrail (scope-check)**: `ScopeGuardrail` is registered on `MeetingAgent` via the agent's guardrail-configuration block. Each function-tool class (`TranscribeTools`, `SummarizeTools`, `DispatchTools`) carries a `Phase` constant. On every tool call the guardrail applies two checks: (A) phase-order check — `TRANSCRIBE` tools require `status ∈ {CREATED, TRANSCRIBING}`; `SUMMARIZE` tools require `status ∈ {TRANSCRIBED, SUMMARIZING}` AND `transcript.isPresent()`; `DISPATCH` tools require `status ∈ {SUMMARIZED, DISPATCHING}` AND `summary.isPresent()`. (B) write-scope check — for `DispatchTools.scheduleFollowUp` and `DispatchTools.createTask`, every attendee/assignee pseudonym in the call arguments must appear in `transcript.segments[].speaker`; any unrecognized pseudonym is a scope-violation. On reject, the guardrail returns a structured error to the agent loop and calls `MeetingEntity.recordScopeRejection(phase, tool, reason)` so the rejection is visible in the UI's rejection-log strip.

## 9. Agent prompts

- `MeetingAgent` → `prompts/meeting-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded meeting `Q2 Engineering Retrospective`; within 90 s the meeting reaches `EVALUATED` with a non-empty transcript, ≥ 2 action items with assignees, and an eval score chip on the card.
2. **J2** — The agent's first iteration on a meeting calls a DISPATCH-phase tool (`scheduleFollowUp`) before `SummaryProduced` has been recorded (mock LLM path). `ScopeGuardrail` rejects the call; a `ScopeRejected` event lands on the entity; the agent retries in-phase; the meeting eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A meeting where the agent attempts to create a task for `[PERSON-99]` (not in the participant list) is rejected by the write-scope check; the rejection event records the pseudonym and reason.
4. **J4** — The PII-redaction audit badge on the transcript panel shows the number of redactions; the raw text containing participant names is never visible in the UI or in any log line from the LLM call.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named meeting-assistant-zoom demonstrating the sequential-pipeline x ops-automation cell. Runs out
of the box (no external services). Maven group io.akka.samples. Maven artifact sequential-pipeline-ops-automation-meeting-assistant-zoom.
Java package io.akka.samples.zoomaimeetingassistant. Akka 3.6.0. HTTP port 9366.

Components to wire (exactly):

- 1 AutonomousAgent MeetingAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/meeting-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  TRANSCRIBE, SUMMARIZE, and DISPATCH tool sets are ALL registered on the agent; phase gating
  and scope checking are the job of ScopeGuardrail, NOT of conditional .tools(...) wiring. The
  before-tool-call guardrail (ScopeGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow MeetingPipelineWorkflow per meetingId with four steps:
  * transcribeStep — runs PiiSanitizer.sanitize(rawTranscript) first (produces
    SanitizedTranscript{text, redactionAudit}), writes MeetingEntity.startTranscribe(redactionAudit),
    then calls componentClient.forAutonomousAgent(MeetingAgent.class,
    "agent-" + meetingId).runSingleTask(
      TaskDef.instructions("Meeting: " + title + "\nPhase: TRANSCRIBE\n" + sanitizedText)
        .metadata("meetingId", meetingId)
        .metadata("phase", "TRANSCRIBE")
        .taskType(MeetingTasks.TRANSCRIBE_MEETING)
    ). Reads forTask(taskId).result(TRANSCRIBE_MEETING) to get Transcript. Writes
    MeetingEntity.recordTranscript(transcript). WorkflowSettings.stepTimeout 90s.
  * summarizeStep — writes SummarizeStarted, then runSingleTask with TaskDef.instructions
    (formatSummarizeContext(transcript, title)) and metadata.phase = "SUMMARIZE", taskType
    SUMMARIZE_MEETING. Writes MeetingEntity.recordSummary(summary). stepTimeout 90s.
  * dispatchStep — writes DispatchStarted, then runSingleTask with TaskDef.instructions
    (formatDispatchContext(summary, transcript, title)) and metadata.phase = "DISPATCH",
    taskType DISPATCH_FOLLOWUPS. Writes MeetingEntity.recordPackage(meetingPackage). stepTimeout 90s.
  * evalStep — runs the deterministic ActionItemScorer over (meetingPackage, summary, transcript)
    and writes MeetingEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(MeetingPipelineWorkflow::error). The error step writes
  MeetingFailed and ends.

- 1 EventSourcedEntity MeetingEntity (one per meetingId). State MeetingRecord{meetingId,
  meetingTitle: Optional<String>, transcript: Optional<Transcript>, summary: Optional<MeetingSummary>,
  meetingPackage: Optional<MeetingPackage>, eval: Optional<EvalResult>, status: MeetingStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. MeetingStatus enum: CREATED, TRANSCRIBING,
  TRANSCRIBED, SUMMARIZING, SUMMARIZED, DISPATCHING, DISPATCHED, EVALUATED, FAILED. Events:
  MeetingCreated{meetingTitle, rawTranscriptEncrypted}, TranscribeStarted{redactionAudit},
  TranscriptProduced{transcript}, SummarizeStarted, SummaryProduced{summary}, DispatchStarted,
  FollowUpsDispatched{meetingPackage}, EvaluationScored{eval}, ScopeRejected{phase, tool, reason},
  MeetingFailed{reason}.
  Commands: create, startTranscribe, recordTranscript, startSummarize, recordSummary,
  startDispatch, recordPackage, recordEvaluation, recordScopeRejection, fail, getMeeting.
  emptyState() returns MeetingRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View MeetingView with row type MeetingRow that mirrors MeetingRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes MeetingEntity events. ONE
  query getAllMeetings: SELECT * AS meetings FROM meeting_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * MeetingEndpoint at /api with POST /meetings (body {meetingTitle, rawTranscript}; mints meetingId;
    calls MeetingEntity.create(meetingTitle, rawTranscript); then starts MeetingPipelineWorkflow with id
    "pipeline-" + meetingId; returns {meetingId}), GET /meetings (list from
    getAllMeetings, sorted newest-first), GET /meetings/{id} (one row), GET
    /meetings/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- MeetingTasks.java declaring three Task<R> constants:
    TRANSCRIBE_MEETING = Task.name("Transcribe meeting").description("Segment the sanitized
      transcript and label each speaker turn").resultConformsTo(Transcript.class);
    SUMMARIZE_MEETING = Task.name("Summarize meeting").description("Extract action items,
      decisions, and a summary paragraph from the transcript").resultConformsTo(MeetingSummary.class);
    DISPATCH_FOLLOWUPS = Task.name("Dispatch follow-ups").description("Schedule follow-up
      calendar events and create tasks from the action items").resultConformsTo(MeetingPackage.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {TRANSCRIBE, SUMMARIZE, DISPATCH}. Each function-tool method is annotated
  with the constant phase (use a custom annotation if the SDK's @FunctionTool does not carry
  a phase field — the guardrail reads it from a parallel registry built at startup if so).

- TranscribeTools.java — @FunctionTool segmentTranscript(String sanitizedText) ->
  List<Segment> reading from src/main/resources/sample-data/transcripts/*.json keyed by
  meetingTitle slug; @FunctionTool labelSpeakers(List<Segment> segments) -> List<Segment>
  reading speaker labels from the matching transcript entry.

- SummarizeTools.java — @FunctionTool extractActionItems(Transcript transcript) ->
  List<ActionItem> (one ActionItem per imperative sentence found in segments); @FunctionTool
  identifyDecisions(Transcript transcript) -> List<Decision> (decisions keyed by "we decided"
  / "agreed to" phrases, decisionId minted as "d-" + sha1(text).substring(0,8)).

- DispatchTools.java — @FunctionTool scheduleFollowUp(List<String> attendeePseudonyms,
  String title, LocalDate scheduledDate) -> FollowUpEvent (returns a deterministic stub entry
  keyed by title slug); @FunctionTool createTask(String assigneePseudonym, String description,
  LocalDate dueDate) -> TaskRecord (deterministic stub).

- ScopeGuardrail.java — implements the before-tool-call hook. Two checks: (A) phase-order
  check using the Phase constant and current MeetingEntity status; (B) write-scope check for
  DispatchTools methods — reads attendeePseudonyms / assigneePseudonym from call arguments,
  looks up transcript.segments[].speaker set from the entity by meetingId, rejects any
  pseudonym not in that set with Guardrail.reject("scope-violation: <pseudonym> is not a
  meeting participant"). On reject ALSO calls MeetingEntity.recordScopeRejection(phase, tool,
  reason).

- PiiSanitizer.java — pure deterministic logic (no LLM). Input: raw transcript String.
  Output: SanitizedTranscript{text, redactionAudit}. Three passes: email regex, phone regex,
  capitalized-bigram NER heuristic. Each distinct match gets a stable pseudonym
  (SHA-1(original).substring(0,4) → index 1..N). Maintains a session-local BiMap<String,String>
  for reverse lookup (used only for the audit display; never exposed to the LLM).

- ActionItemScorer.java — pure deterministic logic (no LLM). Inputs: MeetingPackage, MeetingSummary,
  Transcript. Outputs: EvalResult with score and rationale. Four checks, one point per check
  satisfied, starting from a base of 1: action-item coverage (every actionItem from summary has
  a matching TaskRecord or FollowUpEvent in the package), assignee traceability (every
  TaskRecord.assigneePseudonym appears in transcript.segments[].speaker), event-attendee
  traceability (every FollowUpEvent.attendeePseudonym appears in transcript.segments[].speaker),
  and item parity (actionItems.size() + decisions.size() > 0). Score range 1–5. Rationale
  names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9366 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/meetings.jsonl with 5 seeded meeting lines covering the
  three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/transcripts/*.json — three files keyed by seeded meeting
  title slug, each carrying 8-12 Segment entries with deterministic content so
  TranscribeTools.segmentTranscript returns the same list across restarts. Each file includes
  at least two PII occurrences that PiiSanitizer replaces before the LLM sees the text.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — ops-automation
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = true (transcripts contain
  participant names, emails, phone numbers), decisions.authority_level = recommend-only (the
  package is advisory; humans confirm each follow-up), oversight.human_in_loop = true,
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "out-of-scope-write", "pii-leak-to-llm", "missing-assignee",
  "phase-violation", "unresolved-action-item"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/meeting-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Zoom AI Meeting Assistant", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of meeting cards; right = selected-meeting detail with title header, redaction
  audit badge, transcript segments, summary action items, meeting package, eval-score chip,
  rejection-log strip). Browser title exactly: <title>Akka Sample: Zoom AI Meeting Assistant</title>.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(meetingId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    transcribe-meeting.json — 6 Transcript entries, each with 8-12 Segment items per seeded
      meeting. Each entry's tool_calls array contains 2 calls: 1 segmentTranscript(text) + 1
      labelSpeakers(segments). Plus 1 deliberately PHASE-VIOLATING entry whose tool_calls
      array starts with a scheduleFollowUp(...) call (a DISPATCH-phase tool called during
      TRANSCRIBE) — the guardrail rejects it; the mock then falls through to a normal
      transcribe sequence. The mock selects the violating entry on the FIRST iteration of
      every 3rd meeting (modulo seed) so J2 is reproducible.
    summarize-meeting.json — 6 MeetingSummary entries paired one-to-one with the transcribe
      entries, each with 2-4 ActionItem items and 1-3 Decision items, with tool_calls
      containing extractActionItems + identifyDecisions in order.
    dispatch-followups.json — 6 MeetingPackage entries paired one-to-one. Each carries 1-3
      FollowUpEvent items and matching TaskRecord items, tool_calls containing scheduleFollowUp
      (one per follow-up) + createTask (one per action item). Plus 1 deliberately
      OUT-OF-SCOPE entry whose createTask call carries an assigneePseudonym not in the
      participant list — the guardrail rejects it; J3 verifies this.
- A MockModelProvider.seedFor(meetingId) helper makes per-meeting selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. MeetingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion MeetingTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (transcribeStep
  90s, summarizeStep 90s, dispatchStep 90s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the MeetingRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: MeetingTasks.java with TRANSCRIBE_MEETING, SUMMARIZE_MEETING, DISPATCH_FOLLOWUPS
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9366 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (MeetingAgent). The
  on-decision eval is rule-based (ActionItemScorer.java) and does NOT make an LLM call.
  PiiSanitizer is also rule-based.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  ScopeGuardrail is the runtime mechanism that enforces phase order and write scope. Do NOT
  conditionally register tools per task.
- Task dependency is carried by typed task results: transcribeStep writes Transcript onto the
  entity, summarizeStep reads it and builds the SUMMARIZE task's instruction context from it,
  dispatchStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
