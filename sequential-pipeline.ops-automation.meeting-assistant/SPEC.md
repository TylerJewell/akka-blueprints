# Sample Specification — `meeting-assistant`

This is the natural-language input for `/akka:specify`. Running `/akka:specify @SPEC.md` from inside this folder produces a working Akka project. Sections 1–11 together are the spec; Section 12 chains the rest of the build.

---

## 1. System name + one-line pitch

**Meeting Assistant.** Paste raw meeting notes; the system redacts personal data, extracts action items with the help of an agent, and fans those actions out to Trello cards and a Slack summary message.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern: one meeting moves through ordered stages — sanitize, extract, dispatch — each gated on the previous one completing. The governance pattern is a **PII sanitizer** that redacts the transcript before any model call, plus a **before-tool-call guardrail** that vets every external write (Trello card, Slack message) against an allowlist before it leaves the service. The external writes use the **Real service via env var** integration form: in-process simulators by default, real Trello and Slack when their credentials are present.

## 3. User-facing flows

1. A user pastes meeting notes into the App UI and clicks **Submit**. The response includes the `meetingId`. The UI subscribes to `/api/meetings/sse` and shows the meeting in `RECEIVED`.
2. The pipeline redacts personal data (names, emails, phone numbers) from the transcript. The meeting moves to `SANITIZED`, and the UI shows the redaction count.
3. The agent reads the redacted transcript and returns a summary plus action items. The meeting moves to `EXTRACTED`; the UI lists each action item with its assignee hint.
4. The pipeline creates one Trello card per action item and posts one Slack summary message. The meeting moves to `DISPATCHED`; the UI shows the Trello card links and the Slack message timestamp.
5. If a dispatch target is not on the allowlist, the guardrail refuses the write, the meeting moves to `FAILED`, and the UI shows the refusal reason.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ActionExtractorAgent` | AutonomousAgent | Turn redacted transcript into summary + action items | `PipelineWorkflow` | `MeetingEntity` |
| `ExtractionTasks` | task definitions | `Task<ActionPlan>` constant the agent accepts | — | `ActionExtractorAgent` |
| `MeetingEntity` | EventSourcedEntity | Durable per-meeting lifecycle | `PipelineWorkflow`, `MeetingEndpoint` | `MeetingsView` |
| `InboundNotesQueue` | EventSourcedEntity | Records each submission | `MeetingEndpoint`, `NotesSimulator` | `NotesConsumer` |
| `PipelineWorkflow` | Workflow | `sanitizeStep` → `extractStep` → `dispatchStep` | `NotesConsumer` | `ActionExtractorAgent`, `MeetingEntity`, `SimEndpoint` |
| `MeetingsView` | View | Read model of all meetings | `MeetingEntity` | `MeetingEndpoint` |
| `NotesConsumer` | Consumer | Start a workflow per submission | `InboundNotesQueue` | `PipelineWorkflow` |
| `NotesSimulator` | TimedAction | Drip canned meeting notes every 30s | — | `InboundNotesQueue` |
| `DispatchRetryMonitor` | TimedAction | Retry meetings stuck in `EXTRACTED` every 30s | `MeetingsView` | `PipelineWorkflow` |
| `MeetingEndpoint` | HttpEndpoint | `/api` surface + SSE + metadata | UI, clients | `InboundNotesQueue`, `MeetingsView`, `MeetingEntity` |
| `SimEndpoint` | HttpEndpoint | In-process Trello + Slack receivers | `PipelineWorkflow` | — |
| `AppEndpoint` | HttpEndpoint | Serve `/` redirect and `/app/*` | browser | static-resources |
| `Bootstrap` | service-setup | Schedule TimedActions on startup | — | `NotesSimulator`, `DispatchRetryMonitor` |

## 5. Data model

Pointer: `reference/data-model.md`. Summary:

`Meeting` record (event-sourced state and View row): `String id`, `Optional<String> title`, `MeetingStatus status`, `Optional<Instant> receivedAt`, `Optional<String> sanitizedNotes`, `Optional<Integer> redactionCount`, `Optional<Instant> sanitizedAt`, `Optional<String> summary`, `List<ActionItem> actions` (defaults empty), `Optional<Instant> extractedAt`, `Optional<Instant> dispatchedAt`, `Optional<String> trelloBoardId`, `Optional<String> slackChannel`, `Optional<String> slackMessageTs`, `Optional<String> failureReason`, `Optional<Instant> failedAt`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ActionItem` record: `String title`, `Optional<String> assignee`, `Optional<String> dueHint`, `Optional<String> trelloCardUrl`.

`MeetingStatus` enum: `RECEIVED · SANITIZED · EXTRACTED · DISPATCHED · FAILED`.

Events (`MeetingEvent` sealed interface): `NotesReceived`, `NotesSanitized`, `ActionsExtracted`, `ActionsDispatched`, `DispatchFailed`.

## 6. API contract

Pointer: `reference/api-contract.md`. Top-level surface:

```
POST /api/meetings                 -> { meetingId }
GET  /api/meetings ?status=...     -> { meetings: [Meeting, ...] }
GET  /api/meetings/{id}            -> Meeting
GET  /api/meetings/sse             -> Server-Sent Events of Meeting
POST /sim/trello/cards             -> { cardUrl }      (in-process Trello)
POST /sim/slack/messages           -> { ts }           (in-process Slack)
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Pointer: `reference/ui-mockup.md`. A single self-contained `src/main/resources/static-resources/index.html`. Browser title `<title>Akka Sample: Meeting Assistant</title>`. Five tabs: **Overview** (eyebrow + headline, no subtitle, four cards Try it / How it works / Components / API contract), **Architecture** (the four mermaid diagrams with Lesson 24 CSS overrides), **Risk Survey** (matrix-card style, deployer placeholders muted), **Eval Matrix** (matrix-card style, mechanism pill per row), **App UI** (paste notes, submit, SSE list, per-meeting action items + Trello/Slack results). Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index (Lesson 26); no zombie panels.

## 8. Governance

Pointers: `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **S1 — PII sanitizer.** `sanitizeStep` runs a deterministic redactor over the raw notes (emails, phone numbers, capitalised personal names against a small heuristic) before the agent is called. Emits `NotesSanitized` with the redaction count. The agent never sees raw personal data.
- **G1 — before-tool-call guardrail.** Before each Trello card or Slack message write in `dispatchStep`, a guardrail checks the target Trello board id and Slack channel are on the configured allowlist and that the meeting status is `EXTRACTED`. A non-allowlisted target is refused, emitting `DispatchFailed`.
- **P1 — periodic retry watch.** `DispatchRetryMonitor` re-drives meetings stuck in `EXTRACTED`.
- **A1 — test gate.** Tests pass before deploy.
- **HT1 — operator halt.** An operator can stop accepting new notes.

## 9. Agent prompts

- `ActionExtractorAgent` → `prompts/action-extractor-agent.md`. Purpose: read a redacted transcript and return a structured summary plus a list of action items with assignee hints.

## 10. Acceptance

Pointer: `reference/user-journeys.md`. The journeys that mean "generated correctly":

1. **Submit and extract** — POST notes; within ~30s the meeting reaches `EXTRACTED` with a non-empty `summary` and at least one action item.
2. **Redaction happened** — the `sanitizedNotes` contain no raw email/phone, and `redactionCount` is greater than zero when the input had personal data.
3. **Fan-out** — the meeting reaches `DISPATCHED` with one `trelloCardUrl` per action item and a non-empty `slackMessageTs`.
4. **Guardrail refusal** — a meeting whose configured target is off-allowlist reaches `FAILED` with a refusal `failureReason`; no write reaches `SimEndpoint`.

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md as a whole is the input to `/akka:specify @SPEC.md`.

```
Create a sample named meeting-assistant demonstrating the
sequential-pipeline x ops-automation cell. Integration form: Real service via
env var (in-process Trello/Slack simulators by default; real clients when
TRELLO_API_KEY+TRELLO_TOKEN and SLACK_BOT_TOKEN are set). Maven group
io.akka.samples. Maven artifact meeting-assistant. Java package
io.akka.samples.meetingassistant. Akka 3.6.0. HTTP port 9216.

Components to wire (exactly):
- 1 AutonomousAgent ActionExtractorAgent: reads a redacted transcript, returns
  a typed ActionPlan{summary, List<ActionItem>}. Declares definition() with
  .instructions(...) loaded from prompts/action-extractor-agent.md and
  capability(TaskAcceptance.of(ExtractionTasks.EXTRACT).maxIterationsPerTask(3)).
- ExtractionTasks.java declaring Task<ActionPlan> EXTRACT
  (Task.name("Extract actions").description(...).resultConformsTo(ActionPlan.class)).
- 1 EventSourcedEntity MeetingEntity holding a Meeting record (id, Optional
  title, MeetingStatus enum, and the Optional lifecycle fields in Section 5;
  actions is a non-null List<ActionItem> defaulting to empty). Events:
  NotesReceived, NotesSanitized, ActionsExtracted, ActionsDispatched,
  DispatchFailed. Commands: receive, recordSanitized, recordExtraction,
  recordDispatch, recordFailure, getMeeting. emptyState() returns
  Meeting.initial("") with no commandContext() reference.
- 1 EventSourcedEntity InboundNotesQueue (single instance "default") with one
  command enqueueNotes(title, rawNotes) emitting InboundNotesQueued.
- 1 Workflow PipelineWorkflow with steps sanitizeStep -> extractStep ->
  dispatchStep. sanitizeStep runs PiiRedactor.redact(rawNotes) (deterministic),
  calls MeetingEntity.recordSanitized. extractStep calls
  forAutonomousAgent(ActionExtractorAgent.class,"extract-"+meetingId)
  .runSingleTask(EXTRACT.instructions(redactedNotes)) then
  forTask(taskId).result(EXTRACT); calls MeetingEntity.recordExtraction.
  dispatchStep, for each action item, runs the before-tool-call guardrail then
  POSTs to the Trello client and the Slack client; calls
  MeetingEntity.recordDispatch on success or recordFailure on guardrail refusal.
  Override settings() with stepTimeout(extractStep, 60s) and
  stepTimeout(dispatchStep, 30s); defaultStepRecovery maxRetries(2).
- 1 View MeetingsView with row type Meeting, table updater consuming
  MeetingEntity events. ONE query getAllMeetings SELECT * AS meetings FROM
  meetings_view. No WHERE status filter (Akka cannot auto-index enum columns) —
  filter client-side. Add a streamAllMeetings stream method for SSE.
- 1 Consumer NotesConsumer subscribed to InboundNotesQueue events; on each
  event starts a PipelineWorkflow with a fresh UUID.
- 2 TimedActions: NotesSimulator (every 30s, reads next line from
  src/main/resources/sample-events/meeting-notes.jsonl, calls
  InboundNotesQueue.enqueueNotes); DispatchRetryMonitor (every 30s, queries
  MeetingsView.getAllMeetings, filters EXTRACTED older than 60s, re-runs the
  workflow dispatch).
- 3 HttpEndpoints: MeetingEndpoint at /api (POST meetings, list with
  client-side status filter, single meeting, SSE stream, three
  /api/metadata/* endpoints serving files from src/main/resources/metadata/);
  SimEndpoint at /sim with POST /sim/trello/cards returning {cardUrl} and POST
  /sim/slack/messages returning {ts}; AppEndpoint serving / -> 302
  /app/index.html and /app/* -> static-resources/*.
- TrelloClient and SlackClient helper classes: when TRELLO_API_KEY/SLACK_BOT_TOKEN
  are set, call the real APIs; otherwise POST to the in-process SimEndpoint.

Companion files:
- ActionPlan(String summary, List<ActionItem> actions),
  ActionItem(String title, Optional<String> assignee, Optional<String> dueHint,
  Optional<String> trelloCardUrl).
- PiiRedactor.java: deterministic redaction of emails, phone numbers, and
  capitalised name heuristics; returns redacted text + a count.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9216 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Allowlist keys trello.allowed-board and
  slack.allowed-channel with defaults.
- src/main/resources/sample-events/meeting-notes.jsonl with 8 canned notes,
  at least two containing emails/phone numbers so redaction is observable.
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md}
  (copies for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls S1, G1, P1, A1, HT1 and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking deployer
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per the structure in Section 11 of the
  authoring guide. NO governance-mechanisms section, NO configuration section.
- src/main/resources/static-resources/index.html — one self-contained file
  (no ui/ folder, no npm). Inline CSS + JS; runtime CDN imports for markdown,
  YAML, and mermaid are acceptable. Five tabs per Section 7.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (ActionExtractorAgent -> ActionPlan; see
  src/main/resources/mock-responses/action-extractor-agent.json with 4-6
  entries). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at session end.
- The TRELLO_API_KEY/TRELLO_TOKEN and SLACK_BOT_TOKEN credentials follow the
  same sourcing options. If absent, the in-process simulators are used; the
  service still runs end to end.
- NEVER write any key value to a file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if the model key does not resolve at runtime; never echoes a key.

Mock LLM provider per-agent shapes:
- ActionExtractorAgent -> ActionPlan{ summary: 1-2 sentence string,
  actions: 2-4 ActionItem{ title, assignee (sometimes empty), dueHint
  (sometimes empty), trelloCardUrl empty } }.

Constraints — see AKKA-EXEMPLAR-LESSONS.md. Notably Lessons 1, 4, 6, 7, 8, 9,
10, 11, 12, 13, 23, 24, 25, 26:
- Run command is "/akka:build" (Lesson 9), never "mvn akka:run".
- ActionExtractorAgent extends AutonomousAgent, never downgraded to Agent
  (Lesson 1); it ships ExtractionTasks.java (Lesson 7).
- Workflow steps that call the agent set explicit stepTimeout (Lesson 4).
- Optional<T> for every nullable lifecycle field on the Meeting row (Lesson 6).
- emptyState() never calls commandContext() (Lesson 1/3 family).
- Verify model names are current before locking application.conf (Lesson 8).
- Pick the port from the registry; 9216 here (Lesson 10).
- source.platform is corpus-internal — never user-facing (Lesson 11).
- UI fits the 1080px content column, no horizontal scroll (Lesson 12).
- Integration labels are descriptive, never T1/T2/T3/T4 (Lesson 13).
- No competitor brand names in any user-facing surface (Lesson 23).
- index.html includes the mermaid CSS overrides + theme variables (Lesson 24).
- Tab switching matches by data-tab/data-panel attribute, no zombie panels
  (Lesson 26).
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification, **do not stop and wait for the user**. Continue without prompting:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and the PLAN.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and summarise all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step exceeds 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
