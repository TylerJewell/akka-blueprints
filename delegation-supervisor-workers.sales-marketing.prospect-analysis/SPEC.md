# Sample Specification — `prospect-analysis`

This document is the natural-language input for `/akka:specify`. Running `/akka:specify @SPEC.md` from inside this folder produces a working Akka project. Sections 1–11 together are the spec; Section 12 chains the rest of the workflow.

---

## 1. System name + one-line pitch

**Prospect Analysis Team.** A user submits a target company name; a supervisor delegates to three worker agents that research the company, analyze its org structure to identify decision-makers, and produce an outreach strategy. The user watches each phase complete in the UI.

## 2. What this blueprint demonstrates

The coordination pattern is **delegation-supervisor-workers**: a `ProspectWorkflow` acts as the supervisor and delegates each phase to a dedicated worker agent, recording each typed result into an event-sourced entity before moving to the next worker. The governance pattern wires two mechanisms — a **before-tool-call guardrail** that scopes the workers' web research tool to an allow-list of domains, and a **PII sanitizer** that masks executive contact details before they enter the read model.

## 3. User-facing flows

1. The user enters a company name (and optional primary domain) in the App UI and clicks Analyze. The system creates a prospect in `RESEARCHING` and returns its `prospectId`.
2. The `CompanyResearchAgent` worker gathers a company profile through the scoped research tool. The prospect moves to `ANALYZING` with a populated profile.
3. The `OrgAnalysisAgent` worker derives an org-structure summary and a list of decision-makers; contact hints are masked by the sanitizer. The prospect moves to `STRATEGIZING`.
4. The `OutreachStrategyAgent` worker produces an outreach approach, talking points, and a recommended channel. The prospect moves to `COMPLETED`.
5. If a prospect sits in any in-progress state for more than 3 minutes, the periodic monitor marks it `STALLED` and the UI reflects the change.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CompanyResearchAgent` | AutonomousAgent | Worker: research the target company | `ProspectWorkflow` | returns `CompanyProfile` |
| `OrgAnalysisAgent` | AutonomousAgent | Worker: org structure + decision-makers | `ProspectWorkflow` | returns `OrgAnalysis` |
| `OutreachStrategyAgent` | AutonomousAgent | Worker: outreach strategy | `ProspectWorkflow` | returns `OutreachStrategy` |
| `ProspectWorkflow` | Workflow | Supervisor: delegates each phase, records results | `AnalysisRequestConsumer` | `ProspectEntity`, the three agents |
| `ProspectEntity` | EventSourcedEntity | Prospect lifecycle + state | `ProspectWorkflow`, `StalledAnalysisMonitor` | `ProspectsView` |
| `RequestQueueEntity` | EventSourcedEntity | Inbound work queue | `ProspectEndpoint`, `RequestSimulator` | `AnalysisRequestConsumer` |
| `ProspectsView` | View | CQRS read model over prospect events | `ProspectEntity` | `ProspectEndpoint`, `StalledAnalysisMonitor` |
| `AnalysisRequestConsumer` | Consumer | Starts one workflow per queued company | `RequestQueueEntity` | `ProspectWorkflow` |
| `RequestSimulator` | TimedAction | Drips canned companies every 30s | `sample-events/*.jsonl` | `RequestQueueEntity` |
| `StalledAnalysisMonitor` | TimedAction | Marks stalled prospects every 30s | `ProspectsView` | `ProspectEntity` |
| `ProspectEndpoint` | HttpEndpoint | `/api` REST + SSE + metadata | UI / clients | `RequestQueueEntity`, `ProspectsView`, `ProspectEntity` |
| `AppEndpoint` | HttpEndpoint | Serves the static UI | browser | `static-resources/` |

## 5. Data model

Authoritative — `/akka:implement` writes records exactly as specified. Full detail in `reference/data-model.md`.

- `Prospect` (entity state + view row): `id` (String), `companyName` (String), `domain` (Optional<String>), `status` (`ProspectStatus`), and nullable lifecycle fields all `Optional<T>`: `researchedAt`, `companyProfile`, `analyzedAt`, `orgStructureSummary`, `strategizedAt`, `outreachStrategy`, `completedAt`, `stalledAt`. `decisionMakers` is `List<DecisionMaker>` defaulting to empty.
- `ProspectStatus` enum: `RESEARCHING`, `ANALYZING`, `STRATEGIZING`, `COMPLETED`, `STALLED`.
- Worker result records: `CompanyProfile(String industry, String summary, String hqLocation, int employeeEstimate)`; `OrgAnalysis(String orgStructureSummary, List<DecisionMaker> decisionMakers)`; `DecisionMaker(String name, String title, String department, Optional<String> contactHint)`; `OutreachStrategy(String approach, List<String> talkingPoints, String recommendedChannel)`.
- Events: `ProspectQueued`, `CompanyResearched`, `OrgAnalyzed`, `StrategyDeveloped`, `ProspectStalled`.

## 6. API contract

JSON over HTTP; ACL open (local-dev only). Full schemas in `reference/api-contract.md`.

```
POST /api/analyze                  -> { prospectId }
GET  /api/prospects                -> { prospects: [Prospect, ...] }
GET  /api/prospects/{id}           -> Prospect
GET  /api/prospects/sse            -> Server-Sent Events of Prospect
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Browser title `<title>Akka Sample: Prospect Analysis</title>`. Five tabs: **Overview**, **Architecture**, **Risk Survey**, **Eval Matrix**, **App UI**. Detail in `reference/ui-mockup.md`.

- Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Removed tabs are deleted from the DOM, not hidden.
- The Architecture tab renders the four PLAN.md mermaid diagrams with the Lesson 24 CSS overrides (state-label colour, edge-label `overflow:visible`, `transitionLabelColor #cccccc`).
- The App UI tab: a company-name input and Analyze button, a live SSE list of prospects with per-phase status, and an expandable per-prospect view showing the profile, decision-makers (masked contact hints), and strategy.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Two mechanisms the generated system must wire:

- **G1 — before-tool-call guardrail.** Before any worker invokes the web research tool, the guardrail checks the requested URL/domain against an allow-list; an off-list domain is refused and the call never executes.
- **S1 — PII sanitizer.** Before `OrgAnalysis` results are recorded into `ProspectEntity`, each `DecisionMaker.contactHint` is masked so raw contact details never reach the read model or the UI.

## 9. Agent prompts

- `CompanyResearchAgent` → `prompts/company-research-agent.md` — researches the company through the scoped tool and returns a `CompanyProfile`.
- `OrgAnalysisAgent` → `prompts/org-analysis-agent.md` — derives org structure and decision-makers, returns an `OrgAnalysis`.
- `OutreachStrategyAgent` → `prompts/outreach-strategy-agent.md` — produces an `OutreachStrategy`.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — Analyze a company end-to-end.** Submitting a company drives it through `RESEARCHING` → `ANALYZING` → `STRATEGIZING` → `COMPLETED`, each phase populating its result in the UI.
2. **J2 — Off-domain research blocked.** A research call to a domain outside the allow-list is refused by the guardrail; the prospect does not record that result and the refusal is visible.
3. **J3 — Contact details masked.** Decision-maker contact hints appear masked in the read model and UI, never as raw values.
4. **J4 — Stalled analysis.** A prospect left in an in-progress state past the threshold is marked `STALLED` by the monitor.

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named prospect-analysis demonstrating the
delegation-supervisor-workers × sales-marketing cell. Runs out of the box (no
external services; the web research surface is modeled inside the service).
Maven group io.akka.samples. Maven artifact prospect-analysis. Java package
io.akka.samples.prospectanalysis. Akka 3.6.0. HTTP port 9109.

Components to wire (exactly):
- 3 AutonomousAgents (workers): CompanyResearchAgent (returns CompanyProfile),
  OrgAnalysisAgent (returns OrgAnalysis), OutreachStrategyAgent (returns
  OutreachStrategy). Each extends akka.javasdk.agent.autonomous.AutonomousAgent,
  declares definition() with capability(TaskAcceptance.of(task)
  .maxIterationsPerTask(3)). NEVER downgrade to Agent (Lesson 1).
- 1 ResearchToolEndpoint (HttpEndpoint) modeling the in-process web research
  surface: GET /tools/research?domain=... returns canned JSON profile facts
  from src/main/resources/sample-data/company-facts.json. The research worker's
  tool calls this endpoint; the before-tool-call guardrail (G1) screens the
  requested domain against an allow-list before the call fires.
- 1 Workflow ProspectWorkflow (the supervisor) with steps researchStep ->
  analyzeStep -> strategizeStep. Each step calls forAutonomousAgent(<Worker>
  .class, instanceId).runSingleTask(...) then forTask(taskId).result(...),
  records the typed result into ProspectEntity, and transitions. Override
  settings() with WorkflowSettings.builder().stepTimeout(... ofSeconds(60)) on
  each agent step and defaultStepRecovery(maxRetries(2).failoverTo(error)).
  WorkflowSettings is nested in Workflow — no import (Lesson 5).
- 1 EventSourcedEntity ProspectEntity holding a Prospect record: id, companyName,
  domain (Optional<String>), ProspectStatus enum, decisionMakers (List<
  DecisionMaker>, empty default), and Optional lifecycle fields researchedAt,
  companyProfile, analyzedAt, orgStructureSummary, strategizedAt,
  outreachStrategy, completedAt, stalledAt. Events: ProspectQueued,
  CompanyResearched, OrgAnalyzed, StrategyDeveloped, ProspectStalled. Commands:
  recordResearch, recordAnalysis, recordStrategy, markStalled, getProspect.
  emptyState() returns Prospect.initial("", "") with no commandContext()
  reference (Lesson 3). The recordAnalysis handler applies the S1 sanitizer,
  masking each DecisionMaker.contactHint before the OrgAnalyzed event is stored.
- 1 EventSourcedEntity RequestQueueEntity with one command enqueueAnalysis(
  companyName, domain) emitting AnalysisQueued.
- 1 View ProspectsView with row type Prospect, table updater consuming
  ProspectEntity events. ONE query: getAllProspects SELECT * AS prospects FROM
  prospects_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); filter client-side in callers.
- 1 Consumer AnalysisRequestConsumer subscribed to RequestQueueEntity events;
  on each AnalysisQueued starts a ProspectWorkflow with a fresh UUID.
- 2 TimedActions: RequestSimulator (every 30s, reads next line from
  src/main/resources/sample-events/analysis-requests.jsonl, calls
  RequestQueueEntity.enqueueAnalysis); StalledAnalysisMonitor (every 30s,
  queries ProspectsView.getAllProspects, filters in-progress prospects whose
  last transition was > 3 min ago, calls ProspectEntity.markStalled).
- 2 HttpEndpoints: ProspectEndpoint at /api with analyze, prospects list
  (filter client-side from getAllProspects), single prospect, SSE stream, and
  three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- ProspectTasks.java declaring three Task<R> constants: RESEARCH
  (resultConformsTo CompanyProfile), ANALYZE (OrgAnalysis), STRATEGIZE
  (OutreachStrategy), each Task.name(...).description(...).resultConformsTo(...).
- Records CompanyProfile(String industry, String summary, String hqLocation,
  int employeeEstimate); OrgAnalysis(String orgStructureSummary, List<
  DecisionMaker> decisionMakers); DecisionMaker(String name, String title,
  String department, Optional<String> contactHint); OutreachStrategy(String
  approach, List<String> talkingPoints, String recommendedChannel).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9109 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each
  api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/analysis-requests.jsonl with 8 canned
  company lines.
- src/main/resources/sample-data/company-facts.json with canned per-domain
  facts the research tool returns.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1 and S1 plus a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data classes, capabilities, model family, oversight; marking deployer fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor (Runs out of the box), how to run, the 5 tabs, an
  ASCII architecture diagram, project layout, API contract, license. NO
  governance-mechanisms section, NO configuration section.
- src/main/resources/static-resources/index.html — one self-contained HTML
  file (no ui/, no npm). Inline CSS + JS; runtime CDN imports for markdown and
  YAML are acceptable. Five tabs (Overview / Architecture / Risk Survey / Eval
  Matrix / App UI). Match the governance.html visual style (dark / yellow /
  Instrument Sans / dot-grid). Include the Lesson 24 mermaid CSS overrides and
  theme variables. Tab switching by data-tab / data-panel attribute, never by
  NodeList index (Lesson 26); no hidden zombie panels.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (CompanyResearchAgent -> CompanyProfile, OrgAnalysisAgent ->
  OrgAnalysis, OutreachStrategyAgent -> OutreachStrategy; see
  src/main/resources/mock-responses/{company-research,org-analysis,
  outreach-strategy}.json with 4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes:
- company-research: { industry, summary, hqLocation, employeeEstimate }.
- org-analysis: { orgStructureSummary, decisionMakers:[{name,title,department,
  contactHint}] } — the sanitizer still masks contactHint downstream.
- outreach-strategy: { approach, talkingPoints:[...], recommendedChannel }.

Constraints — see AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout (60s).
- Lesson 6: Optional<T> for every nullable lifecycle field on the view row.
- Lesson 7: each AutonomousAgent has its Task<R> in ProspectTasks.java.
- Lesson 8: verify model names current before locking them.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9109.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the content column with no horizontal scroll.
- Lesson 13: descriptive integration label, never T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label + edge-label CSS overrides in index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: attribute-based tab switching; no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and PLAN.md.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
