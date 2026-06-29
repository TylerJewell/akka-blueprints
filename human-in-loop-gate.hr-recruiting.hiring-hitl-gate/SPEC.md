# SPEC — hiring-hitl-gate

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Human-in-the-Loop Hiring.
**One-line pitch:** A recruiter submits a candidate application; `HiringDecisionProposer` evaluates it and `MeetingProposer` drafts an interview invitation; the workflow pauses at a validator approval gate; a human approves or rejects through the API; on approval `DecisionsReachedService` records the outcome with the accepted proposals.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the hr-recruiting domain: a 3-task graph that proposes a hiring decision, waits at an unassigned validator task that a hiring manager completes through the API, and records the outcome only if approved. The governance pattern is an application-level human approval gate between the proposal phase and the outcome-recording phase, a special-category data sanitizer that scrubs protected attributes before the decision proposal is persisted for review, and a tool-permission guardrail that blocks outcome recording unless the validator has approved.

## 3. User-facing flows

1. A client POSTs a candidate application to `/api/hiring-request`. The response returns `{ applicationId }`. The application appears in the UI in `PROPOSED` once both agents finish (typically 5–30 s), with the hiring proposal and meeting proposal visible.
2. The validator reads the proposals and clicks Approve. This POSTs to `/api/applications/{applicationId}/approve`. The workflow resumes, `DecisionsReachedService` records the outcome, and the application moves to `ACCEPTED` with a decision timestamp.
3. The validator clicks Decline with a reason. This POSTs to `/api/applications/{applicationId}/decline`. The application moves to terminal `DECLINED` and the reason is shown. The outcome-recording step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| HiringDecisionProposer | AutonomousAgent | Evaluates a candidate and returns `HiringProposal{recommendation, rationale}` | HiringWorkflow | ApplicationEntity |
| MeetingProposer | AutonomousAgent | Drafts an interview invitation and returns `MeetingProposal{subject, body, suggestedSlots}` | HiringWorkflow | ApplicationEntity |
| HiringWorkflow | Workflow | Orchestrates propose → await validator → record outcome | HiringEndpoint | HiringDecisionProposer, MeetingProposer, ApplicationEntity |
| ApplicationEntity | EventSourcedEntity | Holds the application state and lifecycle events | HiringWorkflow, HiringEndpoint | ApplicationsView |
| ApplicationsView | View | CQRS read model of all applications | ApplicationEntity | HiringEndpoint |
| HiringEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | HiringWorkflow, ApplicationEntity, ApplicationsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Application` (ApplicationEntity state and ApplicationsView row): `id` (String), `candidateName` (`Optional<String>`), `role` (`Optional<String>`), `status` (ApplicationStatus enum), and lifecycle fields all `Optional<T>`: `submittedAt`, `proposedAt`, `hiringRecommendation`, `hiringRationale`, `meetingSubject`, `meetingBody`, `meetingSuggestedSlots`, `approvedAt`, `approvedBy`, `approverNote`, `declinedAt`, `declinedBy`, `declineReason`, `decidedAt`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ApplicationStatus` enum: `PROPOSED`, `APPROVED`, `DECLINED`, `DECIDED`.

Events: `ApplicationProposed`, `ApplicationApproved`, `ApplicationDeclined`, `OutcomeRecorded`.

Domain records: `HiringProposal(String recommendation, String rationale)`, `MeetingProposal(String subject, String body, String suggestedSlots)`, `ValidatorDecision(String approvedBy, String note)`, `RecordedOutcome(String decidedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/hiring-request                      -> { applicationId }
POST /api/applications/{applicationId}/approve -> 200 | 404
POST /api/applications/{applicationId}/decline -> 200 | 404
GET  /api/applications                         -> { applications: [Application, ...] }
GET  /api/applications/{applicationId}         -> Application
GET  /api/applications/sse                     -> Server-Sent Events of Application
GET  /api/metadata/eval-matrix                 -> text/yaml
GET  /api/metadata/risk-survey                 -> text/yaml
GET  /api/metadata/readme                      -> text/markdown
GET  /                                         -> 302 /app/index.html
GET  /app/{*path}                              -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: Human-in-the-Loop Hiring</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a candidate application, lists applications live via SSE, and shows Approve/Decline buttons on `PROPOSED` applications that have a hiring proposal. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `HiringWorkflow` pauses at the await-validator task; `/api/applications/{id}/approve` and `/api/applications/{id}/decline` resume it.
- **S1 — sanitizer · special-category.** A sanitizer on `HiringDecisionProposer` strips protected-attribute signals (age, gender, ethnicity inferences) from the candidate profile before the proposal is persisted for review.
- **G1 — guardrail · before-tool-call.** A guardrail on `DecisionsReachedService` verifies `ApplicationEntity.status == APPROVED` before the outcome-recording tool runs.

## 9. Agent prompts

- `HiringDecisionProposer` — evaluates a candidate application and proposes a hiring recommendation. See `prompts/hiring-decision-proposer.md`.
- `MeetingProposer` — drafts an interview invitation for an approved candidate. See `prompts/meeting-proposer.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Propose on an application.** POST a candidate application; within ~30 s the application appears in `PROPOSED` with non-empty `hiringRecommendation` and `meetingSubject`.
2. **Approve and record outcome.** Approve a `PROPOSED` application; it reaches `DECIDED` with a non-null `decidedAt` within ~30 s.
3. **Decline a proposal.** Decline a `PROPOSED` application with a reason; it moves to terminal `DECLINED` and the reason shows.
4. **Outcome guard.** The outcome-recording step is never reached for an application that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named hiring-hitl-gate demonstrating the human-in-loop-gate ×
hr-recruiting cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-hr-recruiting-hiring-hitl-gate.
Java package io.akka.samples.humanintheloophiring.
Akka 3.6.0. HTTP port 9636.

Components to wire (exactly):
- 2 AutonomousAgents: HiringDecisionProposer (evaluates a candidate application,
  returns a typed HiringProposal{recommendation,rationale}) and MeetingProposer
  (drafts an interview invitation, returns a typed
  MeetingProposal{subject,body,suggestedSlots}). Each declares definition()
  returning an AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow HiringWorkflow with three tasks: proposeStep -> awaitValidatorStep
  -> recordOutcomeStep. proposeStep calls forAutonomousAgent(HiringDecisionProposer.class,...)
  .runSingleTask(...) and forAutonomousAgent(MeetingProposer.class,...).runSingleTask(...)
  (both tasks run before writing to ApplicationEntity). awaitValidatorStep polls
  ApplicationEntity.getApplication; on PROPOSED it self-schedules a 5-second resume
  timer; on APPROVED it transitions to recordOutcomeStep; on DECLINED it ends.
  recordOutcomeStep writes recordOutcome on ApplicationEntity. Override settings()
  with stepTimeout(60s) on proposeStep and recordOutcomeStep; WorkflowSettings is
  nested in Workflow (no import).
- 1 EventSourcedEntity ApplicationEntity holding an Application record with id,
  candidateName (Optional<String>), role (Optional<String>),
  ApplicationStatus enum {PROPOSED,APPROVED,DECLINED,DECIDED}, and Optional
  lifecycle fields (submittedAt, proposedAt, hiringRecommendation, hiringRationale,
  meetingSubject, meetingBody, meetingSuggestedSlots, approvedAt, approvedBy,
  approverNote, declinedAt, declinedBy, declineReason, decidedAt).
  Events: ApplicationProposed, ApplicationApproved, ApplicationDeclined,
  OutcomeRecorded. Commands: recordProposals, approve, decline, recordOutcome,
  getApplication. emptyState() returns Application.initial("") with no
  commandContext() reference (Lesson 3).
- 1 View ApplicationsView with row type Application, table updater consuming
  ApplicationEntity events. ONE query: getAllApplications SELECT * AS applications
  FROM applications_view. No WHERE status filter (Akka cannot auto-index enum
  columns, Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: HiringEndpoint at /api with hiring-request (starts a
  HiringWorkflow with a fresh UUID), approve, decline, applications list (filter
  client-side from getAllApplications), single application, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- HiringTasks.java declaring two Task<R> constants: HIRE_DECISION (resultConformsTo
  HiringProposal) and MEETING (resultConformsTo MeetingProposal).
- HiringProposal(String recommendation, String rationale),
  MeetingProposal(String subject, String body, String suggestedSlots),
  ValidatorDecision(String approvedBy, String note),
  RecordedOutcome(String decidedAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9636
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (H1, S1, G1) and a
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
  markdown (marked) and YAML (js-yaml) are acceptable. Five tabs: Overview,
  Architecture, Risk Survey, Eval Matrix, App UI. Match the governance.html visual
  style (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (HiringDecisionProposer -> HiringProposal, MeetingProposer -> MeetingProposal;
  see src/main/resources/mock-responses/{hiring-decision-proposer,meeting-proposer}.json
  with 4–6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option a):
- hiring-decision-proposer.json: 4–6 entries, each { "recommendation": "Advance to
  interview", "rationale": "2–3 sentences of plausible evaluation prose" }.
- meeting-proposer.json: 4–6 entries, each { "subject": "Interview invitation string",
  "body": "2–3 sentence invitation body", "suggestedSlots": "e.g. Mon 10am / Tue 2pm" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  HiringTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9636 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
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
