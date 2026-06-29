# SPEC — industry-research-team

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–12 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Industry Research Team.
**One-line pitch:** The user submits a research request ("competitive outlook for grid-scale battery storage"); a supervisor agent splits it into industry sectors, a worker agent analyzes each sector, and the supervisor synthesizes one grounded research brief that the user reads in the App UI tab.

## 2. What this blueprint demonstrates

The delegation supervisor-workers coordination pattern: one supervisor decomposes a request into independent sector sub-tasks, fans them out to a worker agent invoked once per sector, then merges the worker findings into a single result. The governance pattern is a before-agent-response grounding guardrail that blocks any synthesized brief whose claims are not supported by the worker findings, plus a sector sanitizer that appends the required compliance disclaimer to research output before it is released.

## 3. User-facing flows

1. The user submits a research request from the App UI tab (or `POST /api/research-request`). The response carries a `briefId` and the brief appears in `PLANNED` state.
2. The supervisor plans: it returns a brief title and a list of 2–4 sectors. The brief moves to `ANALYZING`.
3. For each sector, the worker agent returns a sector summary plus signal bullets. Each result is recorded; the brief moves to `SYNTHESIZED` once all sectors are in.
4. The supervisor synthesizes a brief body. The grounding guardrail scores it against the worker findings. If the score clears the bar, the brief is released to the sanitizer; otherwise it moves to `BLOCKED` with a reason.
5. The sanitizer appends the sector disclaimer; the brief moves to `COMPLETED` and the sanitized body is shown in the UI.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| SupervisorAgent | AutonomousAgent | Plans sectors from a request; synthesizes the final brief | ResearchWorkflow | ResearchWorkflow |
| SectorAnalystAgent | AutonomousAgent | Worker — analyzes one assigned sector | ResearchWorkflow | ResearchWorkflow |
| ResearchWorkflow | Workflow | Orchestrates plan → analyze → synthesize → sanitize | RequestConsumer | BriefEntity, SupervisorAgent, SectorAnalystAgent |
| BriefEntity | EventSourcedEntity | Holds each brief's lifecycle state and events | ResearchWorkflow | BriefsView |
| InboundRequestQueue | EventSourcedEntity | Buffers inbound requests as events | RequestSimulator, ResearchEndpoint | RequestConsumer |
| BriefsView | View | CQRS read model over BriefEntity events | BriefEntity | ResearchEndpoint |
| RequestConsumer | Consumer | Starts one ResearchWorkflow per queued request | InboundRequestQueue | ResearchWorkflow |
| RequestSimulator | TimedAction | Drips canned requests from a JSONL file every 30s | — | InboundRequestQueue |
| ResearchEndpoint | HttpEndpoint | JSON API: submit, list, single, SSE, metadata | UI / clients | InboundRequestQueue, BriefsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | browser | static-resources |

Names matter — `/akka:specify` uses them verbatim.

## 5. Data model

See `reference/data-model.md`. Entity state record `Brief` carries `String id`, `Optional<String> request`, `BriefStatus status`, `List<String> sectors`, `List<SectorFinding> findings`, and `Optional<T>` lifecycle fields (`plannedAt`, `analyzedAt`, `briefTitle`, `briefBody`, `synthesizedAt`, `groundingScore`, `blockedReason`, `blockedAt`, `sanitizedBody`, `sanitizedAt`, `completedAt`). Status enum: `PLANNED`, `ANALYZING`, `SYNTHESIZED`, `BLOCKED`, `SANITIZED`, `COMPLETED`. Events: `BriefPlanned`, `SectorAnalyzed`, `BriefSynthesized`, `BriefBlocked`, `BriefSanitized`, `BriefCompleted`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

## 6. API contract

See `reference/api-contract.md` for payload schemas. Top-level surface:

```
POST /api/research-request          -> { briefId }
GET  /api/briefs ?status=...        -> { briefs: [Brief, ...] }
GET  /api/briefs/{briefId}          -> Brief
GET  /api/briefs/sse                -> Server-Sent Events of Brief
GET  /api/metadata/eval-matrix      -> text/yaml
GET  /api/metadata/risk-survey      -> text/yaml
GET  /api/metadata/readme           -> text/markdown
GET  /                              -> 302 /app/index.html
GET  /app/{*path}                   -> static-resources/{*path}
```

## 7. UI

Five tabs — Overview / Architecture / Risk Survey / Eval Matrix / App UI — see `reference/ui-mockup.md`. Browser title `<title>Akka Sample: Industry Research Team</title>`. Tab switching matches by `data-tab` → `data-panel` attribute, never by NodeList index, with no hidden zombie panels (Lesson 26). The Architecture tab renders the PLAN.md mermaid diagrams with the state-label CSS overrides and theme variables from Lesson 24. The App UI tab submits a request and shows a live SSE list of briefs with sector chips, grounding score, and the sanitized body.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls wired by the generated system:

- **G1 — grounding guardrail** fires on the supervisor's synthesis (before-agent-response hook). It scores the brief body against the recorded worker findings; a score below the bar moves the brief to `BLOCKED` instead of releasing it.
- **S1 — sector sanitizer** runs in `sanitizeStep` after a brief clears grounding. It appends the configured sector disclaimer and scrubs prohibited advisory phrasing before the brief is marked `COMPLETED`.

## 9. Agent prompts

- `SupervisorAgent` → `prompts/supervisor-agent.md` — plans the sector list, then synthesizes a grounded brief from worker findings.
- `SectorAnalystAgent` → `prompts/sector-analyst-agent.md` — analyzes one assigned sector and returns a summary plus signal bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The journeys that mean the blueprint generated correctly:

1. **Submit and synthesize** — a request produces a `COMPLETED` brief with a non-empty sanitized body and 2–4 sectors within ~60s.
2. **Grounding block** — a synthesis that does not match the worker findings moves the brief to `BLOCKED` with a reason; it is never marked `COMPLETED`.
3. **Disclaimer present** — every `COMPLETED` brief's sanitized body ends with the configured sector disclaimer.
4. **Background load** — the simulator drives a brief from request to `COMPLETED` with no UI interaction.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows. SPEC.md Sections 1–11 are the input to `/akka:specify @SPEC.md`.

```
Create a sample named industry-research-team demonstrating the
delegation-supervisor-workers x research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
industry-research-team. Java package io.akka.samples.industryresearchteam.
Akka 3.6.0. HTTP port 9670.

Components to wire (exactly):
- 2 AutonomousAgents:
  - SupervisorAgent — two task capabilities. PLAN accepts a request string,
    returns ResearchPlan{briefTitle, sectors:List<String>} (2-4 sectors).
    SYNTHESIZE accepts the request plus the list of SectorFinding, returns
    SynthesizedBrief{title, body, groundingScore:double}. definition()
    declares capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)) and
    capability(TaskAcceptance.of(SYNTHESIZE).maxIterationsPerTask(3)).
  - SectorAnalystAgent — worker. ANALYZE accepts {request, sector}, returns
    SectorAnalysis{sector, summary, signals:List<String>}. definition()
    declares capability(TaskAcceptance.of(ANALYZE).maxIterationsPerTask(3)).
- 1 Workflow ResearchWorkflow with steps planStep -> analyzeStep ->
  synthesizeStep -> sanitizeStep. planStep calls
  forAutonomousAgent(SupervisorAgent.class,...).runSingleTask(PLAN...) then
  forTask(taskId).result(...), writes BriefPlanned. analyzeStep iterates the
  planned sectors, calling SectorAnalystAgent once per sector and writing a
  SectorAnalyzed event per result. synthesizeStep calls SupervisorAgent
  SYNTHESIZE with the collected findings; the grounding guardrail evaluates
  the result (before-agent-response) — if groundingScore < 0.6 the step
  writes BriefBlocked and ends; otherwise writes BriefSynthesized.
  sanitizeStep appends the sector disclaimer, writes BriefSanitized then
  BriefCompleted. Override settings() with stepTimeout(60s) on planStep,
  analyzeStep, synthesizeStep; defaultStepRecovery maxRetries(2).failoverTo
  an error step.
- 1 EventSourcedEntity BriefEntity holding the Brief record. Commands:
  recordPlan, recordSectorFinding, recordSynthesis, recordBlock,
  recordSanitization, recordCompletion, getBrief. Events: BriefPlanned,
  SectorAnalyzed, BriefSynthesized, BriefBlocked, BriefSanitized,
  BriefCompleted. emptyState() returns Brief.initial("") with no
  commandContext() reference.
- 1 EventSourcedEntity InboundRequestQueue with one command
  enqueueRequest(request) emitting InboundRequestQueued.
- 1 View BriefsView with row type Brief, table updater consuming BriefEntity
  events. ONE query: getAllBriefs SELECT * AS briefs FROM briefs_view. No
  WHERE status filter (Akka cannot auto-index enum columns) — filter
  client-side in callers.
- 1 Consumer RequestConsumer subscribed to InboundRequestQueue events; on
  each event starts a ResearchWorkflow with a fresh UUID.
- 1 TimedAction RequestSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/research-requests.jsonl and calls
  InboundRequestQueue.enqueueRequest).
- 2 HttpEndpoints: ResearchEndpoint at /api (research-request, briefs list
  filtered client-side from getAllBriefs, single brief, SSE stream, and the
  three /api/metadata/* endpoints serving files from
  src/main/resources/metadata/). AppEndpoint serving / -> 302
  /app/index.html and /app/* -> static-resources/*.

Companion files:
- IndustryResearchTasks.java declaring Task<R> constants: PLAN
  (resultConformsTo ResearchPlan), ANALYZE (SectorAnalysis), SYNTHESIZE
  (SynthesizedBrief).
- Records: ResearchPlan(String briefTitle, List<String> sectors),
  SectorAnalysis(String sector, String summary, List<String> signals),
  SectorFinding(String sector, String summary, List<String> signals),
  SynthesizedBrief(String title, String body, double groundingScore).
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9670 and model-provider blocks for
  anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/research-requests.jsonl with 8 canned
  request lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls G1 and S1 plus the
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data_classes, capabilities, model_family intent, oversight; marking
  jurisdictions, retention, residency, organization fields as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (see Section 12 of the authoring guide).
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS; runtime CDN
  imports for markdown and YAML are acceptable. Five tabs Overview /
  Architecture / Risk Survey / Eval Matrix / App UI. Match the
  governance.html visual style (dark / yellow accent / Instrument Sans /
  dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (SupervisorAgent PLAN -> ResearchPlan with 2-4 sectors;
  SupervisorAgent SYNTHESIZE -> SynthesizedBrief with groundingScore in
  0.6-0.95; SectorAnalystAgent ANALYZE -> SectorAnalysis with 3-5 signals;
  see src/main/resources/mock-responses/{supervisor-agent,sector-analyst-agent}.json
  with 4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in
  .akka-build.yaml; /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at
  session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any key.

Mock LLM provider — per-agent response shapes:
- SupervisorAgent PLAN: ResearchPlan{briefTitle, sectors:[2-4 sector names]}.
- SupervisorAgent SYNTHESIZE: SynthesizedBrief{title, body (a paragraph that
  references each finding), groundingScore (0.60-0.95)}.
- SectorAnalystAgent ANALYZE: SectorAnalysis{sector, summary (2-3 sentences),
  signals:[3-5 bullet strings]}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout(60s).
- Lesson 6: Optional<T> for every nullable lifecycle field on the Brief row.
- Lesson 7: AutonomousAgent ships its companion IndustryResearchTasks.java.
- Lesson 8: verify model names are current before writing them.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9670 in application.conf.
- Lesson 11: no source-platform metadata in any user-facing surface.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box", never T1-T4.
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: mermaid state-label CSS overrides + theme variables in index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute, no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
