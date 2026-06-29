# SPEC — assistloop-support

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** AssistLoop Customer Support.
**One-line pitch:** A customer submits a support ticket; `TriageAgent` drafts a resolution response; the workflow pauses at an approval task; a support supervisor approves or rejects through the API; on approval `ResolutionAgent` delivers the response and closes the ticket.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the cx-support domain: a 3-task graph that triages, then waits at an unassigned approval task that a supervisor completes through the API, then resolves only if approved. The governance pattern is an application-level human approval gate between the triage and resolution phases, a before-tool-call guardrail that blocks the deliver-response tool unless the ticket is approved, and a before-agent-response guardrail on `TriageAgent` that rejects responses below a minimum quality bar before they reach the supervisor queue.

## 3. User-facing flows

1. A client POSTs a ticket to `/api/ticket-request`. The response returns `{ ticketId }`. The ticket appears in the UI in `TRIAGED` once `TriageAgent` finishes (typically 5–30 s), with the drafted summary and response body visible.
2. The supervisor reads the draft response and clicks Approve. This POSTs to `/api/tickets/{ticketId}/approve`. The workflow resumes, `ResolutionAgent` runs, and the confirmation id appears with status `RESOLVED`.
3. The supervisor clicks Reject with a reason. This POSTs to `/api/tickets/{ticketId}/reject`. The ticket moves to terminal `REJECTED` and the reason is shown. The resolution step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| TriageAgent | AutonomousAgent | Analyzes the ticket and drafts a support response; returns `DraftResponse{summary, body}` | SupportWorkflow | TicketEntity |
| ResolutionAgent | AutonomousAgent | Delivers the approved response; returns `DeliveredResolution{confirmationId, deliveredAt}` | SupportWorkflow | TicketEntity |
| SupportWorkflow | Workflow | Orchestrates triage → await approval → resolve | SupportEndpoint | TriageAgent, ResolutionAgent, TicketEntity |
| TicketEntity | EventSourcedEntity | Holds the ticket state and lifecycle events | SupportWorkflow, SupportEndpoint | TicketsView |
| TicketsView | View | CQRS read model of all tickets | TicketEntity | SupportEndpoint |
| SupportEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | SupportWorkflow, TicketEntity, TicketsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Ticket` (TicketEntity state and TicketsView row): `id` (String), `customerMessage` (`Optional<String>`), `status` (TicketStatus enum), and lifecycle fields all `Optional<T>`: `triagedAt`, `responseSummary`, `responseBody`, `approvedAt`, `approvedBy`, `approverNote`, `rejectedAt`, `rejectedBy`, `rejectReason`, `resolvedAt`, `confirmationId`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`TicketStatus` enum: `TRIAGED`, `APPROVED`, `REJECTED`, `RESOLVED`.

Events: `TicketTriaged`, `TicketApproved`, `TicketRejected`, `TicketResolved`.

Domain records: `DraftResponse(String summary, String body)`, `ApprovalDecision(String approvedBy, String note)`, `DeliveredResolution(String confirmationId, String deliveredAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/ticket-request               -> { ticketId }
POST /api/tickets/{ticketId}/approve   -> 200 | 404
POST /api/tickets/{ticketId}/reject    -> 200 | 404
GET  /api/tickets                      -> { tickets: [Ticket, ...] }
GET  /api/tickets/{ticketId}           -> Ticket
GET  /api/tickets/sse                  -> Server-Sent Events of Ticket
GET  /api/metadata/eval-matrix         -> text/yaml
GET  /api/metadata/risk-survey         -> text/yaml
GET  /api/metadata/readme              -> text/markdown
GET  /                                 -> 302 /app/index.html
GET  /app/{*path}                      -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: AssistLoop Customer Support</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a ticket, lists tickets live via SSE, and shows Approve/Reject buttons on `TRIAGED` tickets that have a response body. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `SupportWorkflow` pauses at the await-approval task; `/api/tickets/{id}/approve` and `/api/tickets/{id}/reject` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `ResolutionAgent` verifies `TicketEntity.status == APPROVED` before the simulated deliver-response tool runs.
- **G2 — guardrail · before-agent-response.** A guardrail on `TriageAgent` checks response length and relevance before the draft is persisted for supervisor review.

## 9. Agent prompts

- `TriageAgent` — analyzes a support ticket and drafts a resolution response. See `prompts/triage-agent.md`.
- `ResolutionAgent` — delivers the approved response and returns a confirmation. See `prompts/resolution-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Triage a ticket.** POST a ticket message; within ~30 s a ticket appears in `TRIAGED` with non-empty `responseBody`.
2. **Approve and resolve.** Approve a `TRIAGED` ticket; it reaches `RESOLVED` with a non-null `confirmationId` within ~30 s.
3. **Reject a draft.** Reject a `TRIAGED` ticket with a reason; it moves to terminal `REJECTED` and the reason shows.
4. **Resolution guard.** The resolution step is never reached for a ticket that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named assistloop-support demonstrating the human-in-loop-gate ×
cx-support cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-cx-support-cx-needs-approval.
Java package io.akka.samples.assistloopcustomersupporttemplate.
Akka 3.6.0. HTTP port 9218.

Components to wire (exactly):
- 2 AutonomousAgents: TriageAgent (analyzes the support ticket and drafts a
  response, returns a typed DraftResponse{summary,body}) and ResolutionAgent
  (delivers the approved response, returns a typed
  DeliveredResolution{confirmationId,deliveredAt}). Each declares definition()
  returning an AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow SupportWorkflow with three tasks: triageStep -> awaitApprovalStep
  -> resolveStep. triageStep calls forAutonomousAgent(TriageAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordTriage on
  TicketEntity. awaitApprovalStep polls TicketEntity.getTicket; on TRIAGED it
  self-schedules a 5-second resume timer; on APPROVED it transitions to
  resolveStep; on REJECTED it ends. resolveStep calls ResolutionAgent and writes
  recordResolution. Override settings() with stepTimeout(60s) on triageStep and
  resolveStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity TicketEntity holding a Ticket record with id,
  customerMessage (Optional<String>), TicketStatus enum
  {TRIAGED,APPROVED,REJECTED,RESOLVED}, and Optional lifecycle fields
  (triagedAt, responseSummary, responseBody, approvedAt, approvedBy,
  approverNote, rejectedAt, rejectedBy, rejectReason, resolvedAt,
  confirmationId). Events: TicketTriaged, TicketApproved, TicketRejected,
  TicketResolved. Commands: recordTriage, approve, reject, recordResolution,
  getTicket. emptyState() returns Ticket.initial("") with no commandContext()
  reference (Lesson 3).
- 1 View TicketsView with row type Ticket, table updater consuming TicketEntity
  events. ONE query: getAllTickets SELECT * AS tickets FROM tickets_view. No
  WHERE status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in callers.
- 2 HttpEndpoints: SupportEndpoint at /api with ticket-request (starts a
  SupportWorkflow with a fresh UUID), approve, reject, tickets list (filter
  client-side from getAllTickets), single ticket, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- SupportTasks.java declaring two Task<R> constants: TRIAGE (resultConformsTo
  DraftResponse) and RESOLVE (resultConformsTo DeliveredResolution).
- DraftResponse(String summary, String body), ApprovalDecision(String
  approvedBy, String note), DeliveredResolution(String confirmationId,
  String deliveredAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9218 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1, G2, H1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking jurisdictions,
  declared frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix
  cell, integration descriptor, how to run, the 5 tabs, an ASCII architecture
  diagram, project layout, API contract, license. NO governance-mechanisms
  section. NO configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (TriageAgent -> DraftResponse, ResolutionAgent ->
  DeliveredResolution; see
  src/main/resources/mock-responses/{triage-agent,resolution-agent}.json with
  4–6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in
  .akka-build.yaml; /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option a):
- triage-agent.json: 4–6 entries, each { "summary": "...", "body": "3–5
  sentences of resolution prose" }.
- resolution-agent.json: 4–6 entries, each { "confirmationId":
  "RES-<6-digit>", "deliveredAt": "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  SupportTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9218 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid CSS overrides and theme variables
  (state-label colour, edge-label overflow:visible, transitionLabelColor
  #cccccc).
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
