# SPEC — ambient-email-assistant

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Ambient Email Assistant.
**One-line pitch:** An agent reads incoming email, classifies it, and drafts a reply or schedules a meeting; the workflow pauses at an approval gate; a human approves or dismisses through the API; on approval the agent executes the action (send reply or create calendar event) and closes the thread.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the ops-automation domain: a 4-task graph that triages, drafts an action, waits at an unassigned approval task that a person completes through the API, then executes only if approved. Governance wires four controls: an application-level human approval gate before any outbound action fires, a before-tool-call guardrail that blocks Gmail write operations without approval, a PII sanitizer that strips personally identifiable information from email content before it reaches LLM prompt logs, and an eval-event that records LLM-as-judge scores on each triage and draft decision.

## 3. User-facing flows

1. A client POSTs an incoming email (subject, body, sender) to `/api/email-threads`. The response returns `{ threadId }`. The thread appears in the UI in `TRIAGED` once `TriageAgent` finishes (typically 5–30 s), with the classification category and suggested action visible.
2. The operator reads the classification and the drafted action, then clicks Approve. This POSTs to `/api/threads/{threadId}/approve`. The workflow resumes, the action agent runs (`ReplyDraftAgent` or `MeetingSchedulerAgent`), and the thread transitions to `REPLY_SENT` or `MEETING_SCHEDULED`.
3. The operator clicks Dismiss with an optional note. This POSTs to `/api/threads/{threadId}/dismiss`. The thread moves to terminal `DISMISSED` and the note is shown. No outbound action fires.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| TriageAgent | AutonomousAgent | Classifies an email thread; returns `EmailClassification{category, urgency, suggestedAction}` | EmailWorkflow | EmailThreadEntity |
| ReplyDraftAgent | AutonomousAgent | Drafts an email reply; returns `ReplyDraft{subject, body}` | EmailWorkflow | EmailThreadEntity |
| MeetingSchedulerAgent | AutonomousAgent | Proposes a calendar event; returns `MeetingProposal{title, proposedTime, attendees}` | EmailWorkflow | EmailThreadEntity |
| EmailWorkflow | Workflow | Orchestrates triage → draft action → await approval → execute | EmailEndpoint | TriageAgent, ReplyDraftAgent, MeetingSchedulerAgent, EmailThreadEntity |
| EmailThreadEntity | EventSourcedEntity | Holds the thread state and lifecycle events | EmailWorkflow, EmailEndpoint | ThreadsView |
| ThreadsView | View | CQRS read model of all email threads | EmailThreadEntity | EmailEndpoint |
| EmailEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | EmailWorkflow, EmailThreadEntity, ThreadsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`EmailThread` (EmailThreadEntity state and ThreadsView row): `id` (String), `sender` (`Optional<String>`), `subject` (`Optional<String>`), `status` (ThreadStatus enum), and lifecycle fields all `Optional<T>`: `receivedAt`, `category`, `urgency`, `suggestedAction`, `triageScore`, `draftSubject`, `draftBody`, `proposedMeetingTitle`, `proposedMeetingTime`, `proposedAttendees`, `approvedAt`, `approvedBy`, `approverNote`, `dismissedAt`, `dismissedBy`, `dismissNote`, `actionCompletedAt`, `actionReference`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ThreadStatus` enum: `RECEIVED`, `TRIAGED`, `ACTION_DRAFTED`, `APPROVED`, `REPLY_SENT`, `MEETING_SCHEDULED`, `DISMISSED`.

Events: `ThreadReceived`, `ThreadTriaged`, `ActionDrafted`, `ThreadApproved`, `ThreadDismissed`, `ReplySent`, `MeetingScheduled`.

Domain records: `EmailClassification(String category, String urgency, String suggestedAction)`, `ReplyDraft(String subject, String body)`, `MeetingProposal(String title, String proposedTime, String attendees)`, `ApprovalDecision(String approvedBy, String note)`, `DismissDecision(String dismissedBy, String note)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/email-threads                       -> { threadId }
POST /api/threads/{threadId}/approve          -> 200 | 404
POST /api/threads/{threadId}/dismiss          -> 200 | 404
GET  /api/threads                             -> { threads: [EmailThread, ...] }
GET  /api/threads/{threadId}                  -> EmailThread
GET  /api/threads/sse                         -> Server-Sent Events of EmailThread
GET  /api/metadata/eval-matrix                -> text/yaml
GET  /api/metadata/risk-survey                -> text/yaml
GET  /api/metadata/readme                     -> text/markdown
GET  /                                        -> 302 /app/index.html
GET  /app/{*path}                             -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: Ambient Email Assistant</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits incoming email details, lists threads live via SSE, and shows Approve/Dismiss buttons on `ACTION_DRAFTED` threads. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `EmailWorkflow` pauses at the await-approval task; `/api/threads/{id}/approve` and `/api/threads/{id}/dismiss` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `ReplyDraftAgent` and `MeetingSchedulerAgent` verifies `EmailThreadEntity.status == APPROVED` before any Gmail write or Calendar create tool runs.
- **S1 — sanitizer · pii.** A PII sanitizer strips email addresses, phone numbers, and names from email content before it enters any LLM prompt or is written to logs outside the workflow boundary.
- **E1 — eval-event · on-decision-eval.** An eval event is emitted after each `TriageAgent` and `ReplyDraftAgent` decision, carrying the input, output, and an LLM-as-judge score for downstream quality tracking.

## 9. Agent prompts

- `TriageAgent` — classifies an incoming email thread. See `prompts/triage-agent.md`.
- `ReplyDraftAgent` — drafts a reply for an approved thread. See `prompts/reply-draft-agent.md`.
- `MeetingSchedulerAgent` — proposes a calendar event for a thread classified as meeting-request. See `prompts/meeting-scheduler-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Triage an email.** POST an email; within ~30 s a thread appears in `TRIAGED` with a non-empty `category` and `suggestedAction`.
2. **Approve and send reply.** Approve an `ACTION_DRAFTED` thread with suggestedAction=reply; it reaches `REPLY_SENT` within ~30 s with a non-null `actionReference`.
3. **Approve and schedule meeting.** Approve an `ACTION_DRAFTED` thread with suggestedAction=meeting; it reaches `MEETING_SCHEDULED` within ~30 s.
4. **Dismiss a thread.** Dismiss an `ACTION_DRAFTED` thread with a note; it moves to terminal `DISMISSED` and the note is shown.
5. **Action guard.** The send-email and create-event tools never run on a thread that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named ambient-email-assistant demonstrating the human-in-loop-gate ×
ops-automation cell. Requires a Gmail/Calendar credential for integration (mock
provider available for local testing). Maven group io.akka.samples. Maven artifact
human-in-loop-gate-ops-automation-ambient-email-assistant. Java package
io.akka.samples.ambientemailassistant. Akka 3.6.0. HTTP port 9774.

Components to wire (exactly):
- 3 AutonomousAgents:
  - TriageAgent (classifies email, returns typed EmailClassification{category,urgency,suggestedAction})
  - ReplyDraftAgent (drafts a reply, returns typed ReplyDraft{subject,body})
  - MeetingSchedulerAgent (proposes a calendar event, returns typed
    MeetingProposal{title,proposedTime,attendees})
  Each declares definition() returning an AgentDefinition with .instructions(...)
  loaded from prompts and .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)).
  All three extend akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade
  to Agent.
- 1 Workflow EmailWorkflow with four tasks: triageStep -> draftActionStep ->
  awaitApprovalStep -> executeActionStep. triageStep calls
  forAutonomousAgent(TriageAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...), writes recordTriage on EmailThreadEntity.
  draftActionStep branches on suggestedAction: if "reply" calls ReplyDraftAgent,
  if "meeting" calls MeetingSchedulerAgent; writes recordDraft on entity.
  awaitApprovalStep polls EmailThreadEntity.getThread; on ACTION_DRAFTED it
  self-schedules a 5-second resume timer; on APPROVED it transitions to
  executeActionStep; on DISMISSED it ends. executeActionStep calls the appropriate
  agent action tool (simulated Gmail send / Calendar create) and writes the result.
  Override settings() with stepTimeout(60s) on triageStep, draftActionStep, and
  executeActionStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity EmailThreadEntity holding an EmailThread record with id,
  sender (Optional<String>), subject (Optional<String>), ThreadStatus enum
  {RECEIVED,TRIAGED,ACTION_DRAFTED,APPROVED,REPLY_SENT,MEETING_SCHEDULED,DISMISSED},
  and Optional lifecycle fields (receivedAt, category, urgency, suggestedAction,
  triageScore, draftSubject, draftBody, proposedMeetingTitle, proposedMeetingTime,
  proposedAttendees, approvedAt, approvedBy, approverNote, dismissedAt, dismissedBy,
  dismissNote, actionCompletedAt, actionReference). Events: ThreadReceived,
  ThreadTriaged, ActionDrafted, ThreadApproved, ThreadDismissed, ReplySent,
  MeetingScheduled. Commands: recordTriage, recordDraft, approve, dismiss,
  recordActionComplete, getThread. emptyState() returns EmailThread.initial("") with
  no commandContext() reference (Lesson 3).
- 1 View ThreadsView with row type EmailThread, table updater consuming
  EmailThreadEntity events. ONE query: getAllThreads SELECT * AS threads FROM
  threads_view. No WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: EmailEndpoint at /api with email-threads (starts an
  EmailWorkflow with a fresh UUID), approve, dismiss, threads list (filter
  client-side from getAllThreads), single thread, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- EmailTasks.java declaring three Task<R> constants: TRIAGE (resultConformsTo
  EmailClassification), REPLY_DRAFT (resultConformsTo ReplyDraft), MEETING
  (resultConformsTo MeetingProposal).
- EmailClassification(String category, String urgency, String suggestedAction),
  ReplyDraft(String subject, String body),
  MeetingProposal(String title, String proposedTime, String attendees),
  ApprovalDecision(String approvedBy, String note),
  DismissDecision(String dismissedBy, String note).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9774
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (H1, G1, S1, E1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix cell,
  integration descriptor, how to run, the 5 tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

PII sanitizer — S1 implementation:
- A PiiSanitizer utility class strips email addresses (RFC 5322 pattern), phone
  numbers (E.164 and common US formats), and full names (capitalized word pairs)
  from strings before they are placed in agent prompts or written to event payloads
  outside the workflow.
- Replacements use domain-consistent tokens: [EMAIL], [PHONE], [NAME].
- The sanitizer is invoked by a before-agent-request interceptor on all three agents.

Eval event — E1 implementation:
- An EvalEvent record carries threadId, agentName, inputSummary (sanitized),
  outputSummary, judgeScore (double, 0–1), and evaluatedAt (Instant).
- After each TriageAgent and ReplyDraftAgent task completes, the workflow emits an
  EvalEvent to a dedicated eval topic (logged; integration with an external
  LLM-as-judge service is deployer-configured).

Mock Gmail/Calendar provider (when no credential is available):
- Simulated send: returns a messageId of the form MSG-<uuid>.
- Simulated create event: returns an eventId of the form EVT-<uuid>.
- Mock responses live in src/main/resources/mock-responses/; one JSON file per
  agent (triage-agent.json, reply-draft-agent.json, meeting-scheduler-agent.json)
  with 4–6 entries each.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
      outputs. Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
      /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
      to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- The Gmail/Calendar credential is sourced with the same five-option menu,
  independently of the LLM key.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  EmailTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9774 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Requires Gmail/Calendar credential;
  mock available"), never T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid CSS overrides and theme variables
  (state-label colour, edge-label overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching matches by data-tab/data-panel attribute, never
  NodeList index; no hidden zombie panels in the DOM.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
