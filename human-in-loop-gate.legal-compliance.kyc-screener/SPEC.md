# SPEC — kyc-screener

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** KYC Screener.
**One-line pitch:** A compliance operator submits an entity id and source documents; `ScreenerAgent` assembles the entity file and produces a screening recommendation; the workflow pauses at a compliance-officer approval gate; the officer approves or rejects through the API; on approval `VerdictAgent` closes the case with the final disposition.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the legal-compliance domain: a 3-task graph that screens an entity, then waits at an unassigned approval task that a compliance officer completes through the API, then closes the case only if approved. The governance pattern is an application-level human approval gate between the screening and closing phases, a PII sanitizer that tokenizes entity file data before LLM exposure, and an output guardrail that requires verdicts to cite specific source documents before they reach the reviewer.

## 3. User-facing flows

1. A client POSTs an entity id and source document references to `/api/screening-request`. The response returns `{ caseId }`. The case appears in the UI in `SCREENED` once `ScreenerAgent` finishes (typically 5–30 s), with the entity findings and recommendation visible.
2. The compliance officer reads the screening result and clicks Approve. This POSTs to `/api/cases/{caseId}/approve`. The workflow resumes, `VerdictAgent` runs, and the case transitions to `CLOSED` with disposition `APPROVED`.
3. The compliance officer clicks Reject with a reason. This POSTs to `/api/cases/{caseId}/reject`. The case moves to terminal `REJECTED` and the reason is shown. The close step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ScreenerAgent | AutonomousAgent | Assembles entity file from source documents; returns `ScreeningResult{entityId, findings, recommendation}` | ScreeningWorkflow | CaseEntity |
| VerdictAgent | AutonomousAgent | Closes an approved case; returns `ClosedCase{caseId, disposition, closedAt}` | ScreeningWorkflow | CaseEntity |
| ScreeningWorkflow | Workflow | Orchestrates screen → await approval → close | ScreeningEndpoint | ScreenerAgent, VerdictAgent, CaseEntity |
| CaseEntity | EventSourcedEntity | Holds the case state and lifecycle events | ScreeningWorkflow, ScreeningEndpoint | CasesView |
| CasesView | View | CQRS read model of all cases | CaseEntity | ScreeningEndpoint |
| ScreeningEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | ScreeningWorkflow, CaseEntity, CasesView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Case` (CaseEntity state and CasesView row): `id` (String), `entityId` (`Optional<String>`), `status` (CaseStatus enum), and lifecycle fields all `Optional<T>`: `screenedAt`, `findings`, `recommendation`, `approvedAt`, `approvedBy`, `approverNote`, `rejectedAt`, `rejectedBy`, `rejectReason`, `closedAt`, `disposition`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`CaseStatus` enum: `SCREENED`, `OFFICER_APPROVED`, `REJECTED`, `CLOSED`.

Events: `CaseScreened`, `CaseOfficerApproved`, `CaseRejected`, `CaseClosed`.

Domain records: `ScreeningResult(String entityId, String findings, String recommendation)`, `ApprovalDecision(String approvedBy, String note)`, `ClosedCase(String caseId, String disposition, String closedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/screening-request          -> { caseId }
POST /api/cases/{caseId}/approve     -> 200 | 404
POST /api/cases/{caseId}/reject      -> 200 | 404
GET  /api/cases                      -> { cases: [Case, ...] }
GET  /api/cases/{caseId}             -> Case
GET  /api/cases/sse                  -> Server-Sent Events of Case
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: KYC Screener</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits an entity id and source documents, lists cases live via SSE, and shows Approve/Reject buttons on `SCREENED` cases that have findings. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **S1 — sanitizer · pii.** Entity files contain sensitive PII; a PII sanitizer tokenizes names, ID numbers, and addresses before passing the entity file to `ScreenerAgent`.
- **H1 — hitl · application.** `ScreeningWorkflow` pauses at the await-approval task; `/api/cases/{id}/approve` and `/api/cases/{id}/reject` resume it.
- **G1 — guardrail · before-agent-response.** A guardrail on `ScreenerAgent` checks that the screening result cites at least one specific source document before the findings are persisted for review.

## 9. Agent prompts

- `ScreenerAgent` — assembles an entity file from source documents and produces a screening recommendation with citations. See `prompts/screener-agent.md`.
- `VerdictAgent` — closes an approved case and records the final disposition. See `prompts/verdict-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Screen an entity.** POST an entity id and documents; within ~30 s a case appears in `SCREENED` with non-empty `findings`.
2. **Approve and close.** Approve a `SCREENED` case; it reaches `CLOSED` with disposition `APPROVED` within ~30 s.
3. **Reject a screening result.** Reject a `SCREENED` case with a reason; it moves to terminal `REJECTED` and the reason shows.
4. **Close guard.** The close step is never reached for a case that is not `OFFICER_APPROVED`.

---

## 11. Implementation directives

```
Create a sample named kyc-screener demonstrating the human-in-loop-gate ×
legal-compliance cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-legal-compliance-kyc-screener.
Java package io.akka.samples.kycscreener. Akka 3.6.0. HTTP port 9882.

Components to wire (exactly):
- 2 AutonomousAgents: ScreenerAgent (assembles entity file from source documents,
  returns a typed ScreeningResult{entityId,findings,recommendation}) and
  VerdictAgent (returns a typed ClosedCase{caseId,disposition,closedAt}). Each
  declares definition() returning an AgentDefinition with .instructions(...) loaded
  from prompts and .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)).
  Both extend akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to
  Agent.
- 1 Workflow ScreeningWorkflow with three tasks: screenStep -> awaitApprovalStep
  -> closeStep. screenStep calls forAutonomousAgent(ScreenerAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordScreening on
  CaseEntity. awaitApprovalStep polls CaseEntity.getCase; on SCREENED it
  self-schedules a 5-second resume timer; on OFFICER_APPROVED it transitions to
  closeStep; on REJECTED it ends. closeStep calls VerdictAgent and writes
  recordClose. Override settings() with stepTimeout(60s) on screenStep and
  closeStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity CaseEntity holding a Case record with id, entityId
  (Optional<String>), CaseStatus enum
  {SCREENED,OFFICER_APPROVED,REJECTED,CLOSED}, and Optional lifecycle fields
  (screenedAt, findings, recommendation, approvedAt, approvedBy, approverNote,
  rejectedAt, rejectedBy, rejectReason, closedAt, disposition). Events:
  CaseScreened, CaseOfficerApproved, CaseRejected, CaseClosed. Commands:
  recordScreening, approve, reject, recordClose, getCase. emptyState() returns
  Case.initial("") with no commandContext() reference (Lesson 3).
- 1 View CasesView with row type Case, table updater consuming CaseEntity events.
  ONE query: getAllCases SELECT * AS cases FROM cases_view. No WHERE status filter
  (Akka cannot auto-index enum columns, Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: ScreeningEndpoint at /api with screening-request (starts a
  ScreeningWorkflow with a fresh UUID), approve, reject, cases list (filter
  client-side from getAllCases), single case, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- ScreeningTasks.java declaring two Task<R> constants: SCREEN (resultConformsTo
  ScreeningResult) and CLOSE (resultConformsTo ClosedCase).
- ScreeningResult(String entityId, String findings, String recommendation),
  ApprovalDecision(String approvedBy, String note),
  ClosedCase(String caseId, String disposition, String closedAt).
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9882 and akka.javasdk.agent model-provider
  blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (S1, H1, G1) and a
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

PII sanitizer (S1):
- Before the entity file is passed to ScreenerAgent, a PII sanitizer replaces
  names, national ID numbers, and postal addresses with stable tokens
  (e.g. [NAME_1], [ID_1], [ADDR_1]). A reverse map held in CaseEntity state
  allows the compliance officer view to show original values after approval.
- ScreenerAgent never receives raw PII; it operates on tokenized text only.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ScreenerAgent -> ScreeningResult, VerdictAgent -> ClosedCase; see
  src/main/resources/mock-responses/{screener-agent,verdict-agent}.json with 4–6
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
- screener-agent.json: 4–6 entries, each { "entityId": "...", "findings":
  "2–4 sentences of plausible adverse-media or watchlist findings", "recommendation":
  "PASS | REFER | BLOCK" }.
- verdict-agent.json: 4–6 entries, each { "caseId": "...",
  "disposition": "APPROVED", "closedAt": "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  ScreeningTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9882 declared in application.conf.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
