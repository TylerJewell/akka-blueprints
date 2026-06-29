# Meeting To Tasks — Sample Specification

This file is the natural-language input for `/akka:specify`. Running `/akka:specify @SPEC.md` from inside this folder produces a working Akka project. Sections 1–11 are the spec; Section 12 chains the rest of the workflow.

---

## 1. System name + one-line pitch

**Meeting To Tasks.** A user pastes meeting notes into the App UI. The system extracts a structured task list, holds it for one human approval, then writes the approved tasks to a Trello board, exports a CSV, and posts a Slack notification.

## 2. What this blueprint demonstrates

The coordination pattern is a **sequential pipeline**: a single `Workflow` runs ordered steps — sanitize, extract, await-approval, push-to-board, export-csv, notify — where each step's output feeds the next and the whole run is durable across restarts. The governance pattern wires three controls around the external write surfaces: a **PII sanitizer** that redacts attendee names and emails before any payload leaves the service, a **before-tool-call guardrail** that validates each Trello and Slack payload, and a **human-in-loop approval gate** that holds the extracted tasks until a person approves them. This is the ops-automation domain: turning unstructured meeting text into tracked work items.

## 3. User-facing flows

1. The user opens the App UI tab and pastes meeting notes with a title, then clicks Submit. The system creates a meeting and returns its id. The meeting appears in the live list at status `RECEIVED`.
2. The sanitizer redacts emails and attendee names; the meeting moves to `SANITIZED` with a redaction count.
3. TaskExtractionAgent reads the sanitized notes and returns a task list. The meeting moves to `EXTRACTED`, then `AWAITING_APPROVAL`. The extracted tasks render in the UI with Approve and Reject buttons.
4. The user clicks Approve. The pipeline validates and pushes each task to the Trello board, writes a CSV file, and posts a Slack notification. The meeting moves to `COMPLETED` and shows the board URL, CSV path, and Slack message reference.
5. If the user clicks Reject with a reason, the pipeline stops at `REJECTED` and no external write happens.
6. Without any UI interaction, NotesSimulator drips a canned meeting from a JSONL file every 45 seconds, so the pipeline runs on its own.

These flows are the acceptance journeys in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| TaskExtractionAgent | Agent (request/response) | Turns sanitized notes into a typed `ExtractedTasks` | MeetingPipelineWorkflow | MeetingPipelineWorkflow |
| MeetingPipelineWorkflow | Workflow | Runs the ordered pipeline steps and applies the approval gate | MeetingEndpoint, MeetingsConsumer | MeetingEntity, TaskExtractionAgent, TrelloClient, SlackClient, CsvWriter |
| MeetingEntity | EventSourcedEntity | Holds each meeting's lifecycle state and events | MeetingPipelineWorkflow, MeetingEndpoint | MeetingsView |
| MeetingsView | View | Read model over MeetingEntity events; backs list + SSE | MeetingEntity | MeetingEndpoint |
| NotesSimulator | TimedAction | Drips canned meeting notes from a JSONL file every 45s | (timer) | MeetingEndpoint intake path |
| MeetingEndpoint | HttpEndpoint | JSON + SSE API and metadata endpoints | UI, NotesSimulator | MeetingPipelineWorkflow, MeetingEntity, MeetingsView |
| AppEndpoint | HttpEndpoint | Serves `/` redirect and `/app/*` static UI | browser | static-resources |

Non-primitive collaborators wired by the workflow: `TrelloClient`, `SlackClient`, `CsvWriter` — plain Java helpers. Each external client returns canned results in-process and flips to the real service when its env var is set.

## 5. Data model

Authoritative; `/akka:implement` writes these records verbatim. Full field table in `reference/data-model.md`.

```
record Meeting(
  String id,
  String title,
  String rawNotes,
  MeetingStatus status,
  String receivedAt,
  Optional<String> sanitizedNotes,
  Optional<Integer> redactionCount,
  Optional<List<TaskItem>> extractedTasks,
  Optional<String> approvedAt,
  Optional<String> approvedBy,
  Optional<String> rejectedAt,
  Optional<String> rejectReason,
  Optional<String> trelloBoardUrl,
  Optional<Integer> trelloCardCount,
  Optional<String> csvPath,
  Optional<String> slackMessageTs,
  Optional<String> completedAt,
  Optional<String> failedAt,
  Optional<String> failureReason
)

record TaskItem(String title, String assignee, String dueDate, String priority)
record ExtractedTasks(List<TaskItem> tasks, String summary)

enum MeetingStatus { RECEIVED, SANITIZED, EXTRACTED, AWAITING_APPROVAL, APPROVED, REJECTED, COMPLETED, FAILED }
```

Every nullable lifecycle field is `Optional<T>` (Lesson 6). Events: `MeetingReceived`, `NotesSanitized`, `TasksExtracted`, `MeetingApproved`, `MeetingRejected`, `TasksPushedToBoard`, `CsvExported`, `SlackNotified`, `MeetingCompleted`, `MeetingFailed`. `emptyState()` returns a blank `Meeting` with placeholder identity and no `commandContext()` reference (Lesson 3).

## 6. API contract

Full schemas and SSE event format in `reference/api-contract.md`. Top-level surface:

```
POST /api/meetings                   -> { meetingId }
POST /api/meetings/{id}/approve      -> 200 | 404
POST /api/meetings/{id}/reject       -> 200 | 404
GET  /api/meetings ?status=...       -> { meetings: [Meeting, ...] }
GET  /api/meetings/{id}              -> Meeting
GET  /api/meetings/sse               -> Server-Sent Events of Meeting
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

The `?status=` filter is applied client-side in the endpoint over `getAllMeetings`; the view has no enum WHERE clause (Lesson 2).

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm). Browser title `<title>Akka Sample: Meeting To Tasks</title>`. Five tabs: **Overview**, **Architecture**, **Risk Survey**, **Eval Matrix**, **App UI**. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, and removed panels are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the `PLAN.md` mermaid diagrams with the state-label colour and edge-label `overflow:visible` CSS overrides plus theme variables from Lesson 24. The Risk Survey and Eval Matrix tabs render in the `matrix-card` / `matrix-row` style. The App UI tab: a notes textarea + title field + Submit; a live SSE list of meetings; per-meeting Approve/Reject buttons when status is `AWAITING_APPROVAL`. Full layout in `reference/ui-mockup.md`.

## 8. Governance

Controls in `eval-matrix.yaml`; deployer survey in `risk-survey.yaml`. Three mechanisms the generated system wires:

- **S1 — sanitizer (pii).** A sanitize step runs before extraction and before any external write, redacting emails and attendee names from the notes and from each task payload. The redaction count is recorded on the entity.
- **G1 — guardrail (before-tool-call).** Before each Trello and Slack write, a guardrail validates the payload (required fields present, no un-redacted PII, board/channel ids well-formed). A failing payload blocks the write and fails the step.
- **H1 — hitl (application).** The workflow pauses at `AWAITING_APPROVAL`; the external writes only happen after a human POSTs approve.

## 9. Agent prompts

- `prompts/task-extraction-agent.md` — system prompt for TaskExtractionAgent: extract a structured list of action items (title, assignee, due date, priority) from sanitized meeting notes and return typed `ExtractedTasks`.

## 10. Acceptance

Inline summary; full journeys in `reference/user-journeys.md`.

- **J1** — Submit notes via the API; a meeting reaches `AWAITING_APPROVAL` with a non-empty task list within ~30s.
- **J2** — Approve a meeting; it reaches `COMPLETED` with a non-null board URL, CSV path, and Slack reference.
- **J3** — Reject a meeting; it reaches `REJECTED` and no external write fires.
- **J4** — Submit notes containing an email and an attendee name; the sanitized notes and the pushed payloads contain neither, and `redactionCount > 0`.

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named meeting-to-tasks demonstrating the
sequential-pipeline x ops-automation cell. Integration form: real service via
env var (Trello + Slack write surfaces default to in-process facades, flip to
real clients when TRELLO_API_KEY/TRELLO_TOKEN/SLACK_BOT_TOKEN are set).
Maven group io.akka.samples. Maven artifact meeting-to-tasks. Java package
io.akka.samples.meetingtotasks. Akka 3.6.0. HTTP port 9450.

Components to wire (exactly):
- 1 Agent TaskExtractionAgent (request/response — extends akka.javasdk.agent.Agent,
  NOT AutonomousAgent). One method extract(String sanitizedNotes) returning
  Effect<ExtractedTasks> via effects().systemMessage(...).userMessage(...)
  .responseAs(ExtractedTasks.class).thenReply(). No Tasks.java companion (that
  is only for AutonomousAgent). Do not silently swap the base class (Lesson 1).
- 1 Workflow MeetingPipelineWorkflow with steps sanitizeStep -> extractStep ->
  awaitApprovalStep -> pushToBoardStep -> exportCsvStep -> notifyStep. sanitizeStep
  redacts emails/attendee-names and records redactionCount via
  MeetingEntity.recordSanitized. extractStep calls
  componentClient.forAgent().inSession(meetingId).method(TaskExtractionAgent::extract)
  .invoke(notes) then MeetingEntity.recordExtraction. awaitApprovalStep polls
  MeetingEntity.getMeeting; on AWAITING_APPROVAL self-schedules a 5-second resume
  timer; on APPROVED transitions to pushToBoardStep; on REJECTED ends. pushToBoardStep
  runs the before-tool-call guardrail then calls TrelloClient.createCards.
  exportCsvStep calls CsvWriter.write. notifyStep runs the guardrail then
  SlackClient.postMessage, then MeetingEntity.recordCompletion. Override settings()
  with stepTimeout(MeetingPipelineWorkflow::extractStep, ofSeconds(60)) and
  defaultStepRecovery(maxRetries(2).failoverTo(MeetingPipelineWorkflow::error)).
  WorkflowSettings is nested in Workflow — no import (Lesson 5).
- 1 EventSourcedEntity MeetingEntity holding the Meeting record with id, title,
  rawNotes, MeetingStatus enum, receivedAt, and 14 Optional lifecycle fields
  (sanitizedNotes, redactionCount, extractedTasks, approvedAt, approvedBy,
  rejectedAt, rejectReason, trelloBoardUrl, trelloCardCount, csvPath,
  slackMessageTs, completedAt, failedAt, failureReason). Events: MeetingReceived,
  NotesSanitized, TasksExtracted, MeetingApproved, MeetingRejected,
  TasksPushedToBoard, CsvExported, SlackNotified, MeetingCompleted, MeetingFailed.
  Commands: receive, recordSanitized, recordExtraction, approve, reject,
  recordBoardPush, recordCsv, recordSlack, recordCompletion, markFailed,
  getMeeting. emptyState() returns a blank Meeting with placeholder id and no
  commandContext() reference (Lesson 3).
- 1 View MeetingsView with row type Meeting, table updater consuming MeetingEntity
  events. ONE query: getAllMeetings SELECT * AS meetings FROM meetings_view. No
  WHERE status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in the endpoint.
- 1 TimedAction NotesSimulator (every 45s) reading the next line from
  src/main/resources/sample-events/meeting-notes.jsonl and POSTing it through the
  same intake path the endpoint uses (start a MeetingPipelineWorkflow with a fresh
  UUID).
- 2 HttpEndpoints: MeetingEndpoint at /api (POST meetings, approve, reject,
  meetings list filtered client-side from getAllMeetings, single meeting, SSE
  stream via serverSentEventsForView, and three /api/metadata/* endpoints serving
  the YAML/MD files from src/main/resources/metadata/). AppEndpoint serving / ->
  302 /app/index.html and /app/* -> static-resources/*.
- Helper classes (not Akka primitives): TrelloClient.createCards(boardId, tasks),
  SlackClient.postMessage(channel, text), CsvWriter.write(meetingId, tasks),
  PiiSanitizer.redact(text). TrelloClient and SlackClient return canned in-process
  results unless TRELLO_API_KEY/TRELLO_TOKEN/SLACK_BOT_TOKEN are present, in which
  case they call the real HTTP APIs.

Companion files:
- ExtractedTasks(List<TaskItem> tasks, String summary),
  TaskItem(String title, String assignee, String dueDate, String priority),
  ApprovalDecision(String approvedBy).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9450
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Verify
  model names are current before locking (Lesson 8).
- src/main/resources/sample-events/meeting-notes.jsonl with 8 canned meeting-note
  lines, each containing at least one email and one attendee name so the sanitizer
  has work to do.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root files for the endpoint to serve from the classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1, S1, H1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor, how to run, the tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section, NO
  configuration section (Lesson 20).
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown/YAML libs
  are acceptable. Five tabs: Overview, Architecture (renders PLAN.md mermaid with
  the Lesson 24 CSS overrides + theme variables), Risk Survey (matrix-card style,
  mutes TO_BE_COMPLETED_BY_DEPLOYER), Eval Matrix (matrix-card style, colored
  mechanism pill in the label column), App UI (submit notes, SSE list, Approve/
  Reject on AWAITING_APPROVAL). Tab switching by data-tab/data-panel attribute,
  no zombie panels (Lesson 26). Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for ANTHROPIC_API_KEY
  / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (TaskExtractionAgent -> ExtractedTasks; see
  src/main/resources/mock-responses/task-extraction-agent.json with 4-6 entries).
  Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources it before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- The same five options apply to the Trello/Slack credentials (TRELLO_API_KEY,
  TRELLO_TOKEN, SLACK_BOT_TOKEN). If unset, the clients stay on their in-process
  facades and the pipeline still completes end-to-end.
- NEVER write any key value to any file Akka creates. Record only the reference —
  env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; it never echoes captured key material.

Mock LLM provider (per-agent shapes for option a):
- TaskExtractionAgent -> ExtractedTasks{ tasks: [3-5 TaskItem each with a plausible
  title, an assignee first name, a due date within 14 days, a priority of
  low/medium/high], summary: one sentence }. 4-6 canned variants.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: TaskExtractionAgent extends Agent (request/response); never silently
  downgrade or upgrade the AI primitive.
- Lesson 4: every workflow step calling the agent overrides stepTimeout to 60s.
- Lesson 6: Optional<T> for every nullable field on the Meeting row record.
- Lesson 7: only an AutonomousAgent needs Tasks.java; this sample uses Agent, so
  no Tasks.java is generated.
- Lesson 8: verify model names are current before writing application.conf.
- Lesson 9: the run command is "/akka:build" (Claude Code slash command), never
  "mvn akka:run".
- Lesson 10: explicit dev-mode http-port = 9450.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column with no horizontal scroll.
- Lesson 13: integration labels are descriptive ("Real service via env var"),
  never T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes the mermaid state-label CSS overrides, edge-label
  foreignObject overflow:visible, and transitionLabelColor #cccccc.
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and the PLAN.md.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9450/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — a missing model key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
