# SPEC — akka-hitl-gate

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** HITL Ops Automation Gate.
**One-line pitch:** An operator submits an ops task request; `ProposalAgent` analyzes it and drafts an action proposal; the workflow pauses at an approval gate; a human approves or rejects through the API; on approval `ExecutionAgent` carries out the action and returns the outcome.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the ops-automation domain: a 3-task graph that proposes, then waits at an unassigned approval task that a person completes through the API, then executes only if approved. The governance pattern is an application-level human approval gate between the proposal and execution phases, an output guardrail before a proposal reaches the reviewer, and a tool-permission guardrail that blocks the execution step unless the action is approved.

## 3. User-facing flows

1. An operator POSTs an ops task description to `/api/action-request`. The response returns `{ actionId }`. The action appears in the UI in `PROPOSED` once `ProposalAgent` finishes (typically 5–30 s), with the proposed summary and estimated impact visible.
2. The reviewer reads the proposal and clicks Approve. This POSTs to `/api/actions/{actionId}/approve`. The workflow resumes, `ExecutionAgent` runs, and the outcome appears with status `EXECUTED`.
3. The reviewer clicks Reject with a reason. This POSTs to `/api/actions/{actionId}/reject`. The action moves to terminal `REJECTED` and the reason is shown. The execution step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ProposalAgent | AutonomousAgent | Analyzes an ops task; returns `ActionProposal{summary, estimatedImpact, actionPayload}` | GateWorkflow | ActionEntity |
| ExecutionAgent | AutonomousAgent | Executes an approved action; returns `ActionResult{outcome, executedAt}` | GateWorkflow | ActionEntity |
| GateWorkflow | Workflow | Orchestrates propose → await approval → execute | GateEndpoint | ProposalAgent, ExecutionAgent, ActionEntity |
| ActionEntity | EventSourcedEntity | Holds the action state and lifecycle events | GateWorkflow, GateEndpoint | ActionsView |
| ActionsView | View | CQRS read model of all actions | ActionEntity | GateEndpoint |
| GateEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | GateWorkflow, ActionEntity, ActionsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Action` (ActionEntity state and ActionsView row): `id` (String), `taskDescription` (`Optional<String>`), `status` (ActionStatus enum), and lifecycle fields all `Optional<T>`: `proposedAt`, `proposalSummary`, `estimatedImpact`, `actionPayload`, `approvedAt`, `approvedBy`, `approverNote`, `rejectedAt`, `rejectedBy`, `rejectReason`, `executedAt`, `outcome`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ActionStatus` enum: `PROPOSED`, `APPROVED`, `REJECTED`, `EXECUTED`.

Events: `ActionProposed`, `ActionApproved`, `ActionRejected`, `ActionExecuted`.

Domain records: `ActionProposal(String summary, String estimatedImpact, String actionPayload)`, `ApprovalDecision(String approvedBy, String note)`, `ActionResult(String outcome, String executedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/action-request               -> { actionId }
POST /api/actions/{actionId}/approve   -> 200 | 404
POST /api/actions/{actionId}/reject    -> 200 | 404
GET  /api/actions                      -> { actions: [Action, ...] }
GET  /api/actions/{actionId}           -> Action
GET  /api/actions/sse                  -> Server-Sent Events of Action
GET  /api/metadata/eval-matrix         -> text/yaml
GET  /api/metadata/risk-survey         -> text/yaml
GET  /api/metadata/readme              -> text/markdown
GET  /                                 -> 302 /app/index.html
GET  /app/{*path}                      -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: HITL Ops Automation Gate</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits an ops task request, lists actions live via SSE, and shows Approve/Reject buttons on `PROPOSED` actions that have a summary. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `GateWorkflow` pauses at the await-approval task; `/api/actions/{id}/approve` and `/api/actions/{id}/reject` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `ExecutionAgent` verifies `ActionEntity.status == APPROVED` before the simulated execution tool runs.
- **G2 — guardrail · before-agent-response.** A guardrail on `ProposalAgent` checks proposal completeness (non-empty summary, non-empty actionPayload) before the proposal is persisted for review.

## 9. Agent prompts

- `ProposalAgent` — analyzes an ops task and produces an action proposal. See `prompts/proposal-agent.md`.
- `ExecutionAgent` — executes an approved action and returns its outcome. See `prompts/execution-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Propose an action.** POST a task description; within ~30 s an action appears in `PROPOSED` with non-empty `proposalSummary` and `actionPayload`.
2. **Approve and execute.** Approve a `PROPOSED` action; it reaches `EXECUTED` with a non-null `outcome` within ~30 s.
3. **Reject a proposal.** Reject a `PROPOSED` action with a reason; it moves to terminal `REJECTED` and the reason shows.
4. **Execution guard.** The execution step is never reached for an action that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named akka-hitl-gate demonstrating the human-in-loop-gate ×
ops-automation cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-ops-automation-akka-hitl-gate.
Java package io.akka.samples.humanintheloop. Akka 3.6.0. HTTP port 9892.

Components to wire (exactly):
- 2 AutonomousAgents: ProposalAgent (analyzes an ops task, returns a typed
  ActionProposal{summary, estimatedImpact, actionPayload}) and ExecutionAgent
  (returns a typed ActionResult{outcome, executedAt}). Each declares definition()
  returning an AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow GateWorkflow with three tasks: proposeStep -> awaitApprovalStep
  -> executeStep. proposeStep calls forAutonomousAgent(ProposalAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordProposal on
  ActionEntity. awaitApprovalStep polls ActionEntity.getAction; on PROPOSED it
  self-schedules a 5-second resume timer; on APPROVED it transitions to
  executeStep; on REJECTED it ends. executeStep calls ExecutionAgent and writes
  recordExecution. Override settings() with stepTimeout(60s) on proposeStep and
  executeStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity ActionEntity holding an Action record with id,
  taskDescription (Optional<String>), ActionStatus enum
  {PROPOSED,APPROVED,REJECTED,EXECUTED}, and Optional lifecycle fields
  (proposedAt, proposalSummary, estimatedImpact, actionPayload, approvedAt,
  approvedBy, approverNote, rejectedAt, rejectedBy, rejectReason, executedAt,
  outcome). Events: ActionProposed, ActionApproved, ActionRejected,
  ActionExecuted. Commands: recordProposal, approve, reject, recordExecution,
  getAction. emptyState() returns Action.initial("") with no commandContext()
  reference (Lesson 3).
- 1 View ActionsView with row type Action, table updater consuming ActionEntity
  events. ONE query: getAllActions SELECT * AS actions FROM actions_view. No WHERE
  status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in callers.
- 2 HttpEndpoints: GateEndpoint at /api with action-request (starts a
  GateWorkflow with a fresh UUID), approve, reject, actions list (filter
  client-side from getAllActions), single action, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- GateTasks.java declaring two Task<R> constants: PROPOSE (resultConformsTo
  ActionProposal) and EXECUTE (resultConformsTo ActionResult).
- ActionProposal(String summary, String estimatedImpact, String actionPayload),
  ApprovalDecision(String approvedBy, String note),
  ActionResult(String outcome, String executedAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9892
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (G1, G2, H1) and a
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

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ProposalAgent -> ActionProposal, ExecutionAgent -> ActionResult; see
  src/main/resources/mock-responses/{proposal-agent,execution-agent}.json with 4–6
  entries each). Sets model-provider = mock.
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
- proposal-agent.json: 4–6 entries, each { "summary": "...", "estimatedImpact":
  "...", "actionPayload": "..." }.
- execution-agent.json: 4–6 entries, each { "outcome": "...",
  "executedAt": "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  GateTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9892 declared in application.conf.
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
