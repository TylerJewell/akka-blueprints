# SPEC — marketing-strategy-team

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–12 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Marketing Strategy Team.
**One-line pitch:** A user submits a project brief; a lead strategist agent delegates research, strategy, and content work to specialist agents, then assembles one grounded marketing plan with a self-evaluation score.

## 2. What this blueprint demonstrates

The delegation-supervisor-workers coordination pattern: one supervisor agent breaks a brief into assignments, three worker agents each produce a typed artifact, and the supervisor assembles a final plan. The governance pattern wires a before-agent-response guardrail that holds the assembled plan to claims grounded in the research findings, and an eval-event that runs a self-evaluation of the plan against the original brief.

## 3. User-facing flows

1. The user opens the App UI tab and types a project brief (product, audience, goal). Submit returns a `campaignId`; the campaign appears in `RECEIVED`.
2. The `StrategyWorkflow` runs: the campaign moves through `RESEARCHED`, `STRATEGIZED`, `CONTENT_DRAFTED`, then `ASSEMBLED`.
3. When the plan assembles, a self-evaluation runs and the campaign moves to `EVALUATED` with a score against the brief. The UI shows the plan sections and the score.
4. Without any interaction, the `BriefSimulator` drips a fresh brief on an interval, so campaigns keep arriving.

These flows become the acceptance journeys in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| CampaignLeadAgent | AutonomousAgent | Supervisor — breaks the brief into assignments, assembles the plan, runs self-eval | StrategyWorkflow | CampaignEntity |
| MarketResearchAgent | AutonomousAgent | Worker — calls the web-search tool, returns grounded findings | StrategyWorkflow | WebSearchEndpoint |
| StrategyAgent | AutonomousAgent | Worker — returns positioning, channels, tactics | StrategyWorkflow | — |
| ContentAgent | AutonomousAgent | Worker — returns content artifacts (taglines, post copy) | StrategyWorkflow | — |
| StrategyWorkflow | Workflow | Delegates research → strategy → content → assemble + eval | BriefConsumer | CampaignEntity, agents |
| CampaignEntity | EventSourcedEntity | Durable per-campaign lifecycle | StrategyWorkflow | CampaignsView |
| BriefQueue | EventSourcedEntity | Records each inbound brief | BriefSimulator, CampaignEndpoint | BriefConsumer |
| CampaignsView | View | List-of-campaigns read model | CampaignEntity | CampaignEndpoint |
| BriefConsumer | Consumer | Starts a workflow per inbound brief | BriefQueue | StrategyWorkflow |
| BriefSimulator | TimedAction | Drips a canned brief on an interval | sample-events | BriefQueue |
| WebSearchEndpoint | HttpEndpoint | In-process grounding source — returns canned search results | MarketResearchAgent | — |
| CampaignEndpoint | HttpEndpoint | `/api` surface + SSE + metadata | UI / clients | BriefQueue, CampaignsView, CampaignEntity |
| AppEndpoint | HttpEndpoint | Serves `/` and `/app/*` | browser | static-resources |
| Bootstrap | service-setup | Schedules the simulator on startup | — | BriefSimulator |

Names are used verbatim by `/akka:specify`.

## 5. Data model

Authoritative record forms are in `reference/data-model.md`. Summary:

`Campaign` (event-sourced state and View row) has `String id`, `Optional<String> brief`, `CampaignStatus status`, and `Optional<T>` lifecycle fields: `receivedAt`, `findings`, `researchedAt`, `strategy`, `strategizedAt`, `content`, `contentDraftedAt`, `planSummary`, `assembledAt`, `evalScore` (`Optional<Integer>`), `evalNotes`, `evaluatedAt`, `failureReason`, `failedAt`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`CampaignStatus` enum: `RECEIVED · RESEARCHED · STRATEGIZED · CONTENT_DRAFTED · ASSEMBLED · EVALUATED · FAILED`.

`CampaignEvent` (sealed): `BriefReceived`, `ResearchRecorded`, `StrategyRecorded`, `ContentRecorded`, `PlanAssembled`, `PlanEvaluated`, `CampaignFailed`.

Worker result records: `MarketFindings(List<String> claims, List<String> sources)`, `StrategyDraft(String positioning, List<String> channels, List<String> tactics)`, `ContentArtifacts(List<String> taglines, List<String> posts)`, `CampaignPlan(String summary)`, `PlanEval(int score, String notes)`.

## 6. API contract

Full schemas in `reference/api-contract.md`. Top-level surface:

```
POST /api/briefs                         -> { campaignId }
GET  /api/campaigns ?status=...          -> { campaigns: [Campaign, ...] }
GET  /api/campaigns/{id}                 -> Campaign
GET  /api/campaigns/sse                  -> Server-Sent Events of Campaign
GET  /api/search ?q=...                  -> { results: [SearchResult, ...] }  (in-process grounding source)
GET  /api/metadata/{readme,risk-survey,eval-matrix} -> text
GET  /                                   -> 302 /app/index.html
GET  /app/{*path}                        -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build (Lesson 17). Browser title `<title>Akka Sample: Marketing Strategy Team</title>`. Five tabs — Overview / Architecture / Risk Survey / Eval Matrix / App UI — described in `reference/ui-mockup.md`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Architecture tab mermaid diagrams carry the state-label CSS overrides and theme variables from Lesson 24. The App UI tab submits a brief and shows a live SSE list of campaigns; each assembled campaign shows its plan sections, grounded claims, and self-eval score.

## 8. Governance

Controls are in `eval-matrix.yaml`; the deployer survey is in `risk-survey.yaml`. Two mechanisms the generated system wires:

- **G1 — claim-grounding guardrail (guardrail · before-agent-response):** before `CampaignLeadAgent`'s assembled plan is persisted, a guardrail checks that the plan's factual claims are present in the recorded `MarketFindings`; ungrounded plans are blocked and the workflow retries the assemble step.
- **E1 — self-evaluation (eval-event · on-decision-eval):** when the plan assembles, the workflow runs a self-evaluation pass scoring the plan against the original brief and records a `PlanEvaluated` event with the score and notes.

## 9. Agent prompts

- `prompts/campaign-lead-agent.md` — supervisor; assigns work, assembles the plan, self-evaluates.
- `prompts/market-research-agent.md` — worker; grounds findings in web-search results.
- `prompts/strategy-agent.md` — worker; produces positioning, channels, tactics.
- `prompts/content-agent.md` — worker; produces taglines and post copy.

## 10. Acceptance

Inlined from `reference/user-journeys.md`:

1. **J1 — Submit a brief, watch a plan assemble.** POST `/api/briefs`; within ~60 s the campaign reaches `ASSEMBLED` then `EVALUATED` with a non-empty plan summary.
2. **J2 — Grounded claims only.** The assembled plan's claims are a subset of the recorded `MarketFindings`; an ungrounded draft is blocked by G1 and re-assembled.
3. **J3 — Self-eval score.** The `EVALUATED` campaign carries an integer `evalScore` and `evalNotes` referencing the brief.
4. **J4 — Background load.** With no UI interaction, `BriefSimulator` seeds a brief and a campaign runs end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows. The whole SPEC.md — Sections 1–12 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named marketing-strategy-team demonstrating the
delegation-supervisor-workers x sales-marketing cell. Runs out of the box (the
web-search grounding source is modeled in-process). Maven group
io.akka.samples. Maven artifact marketing-strategy-team. Java package
io.akka.samples.marketingstrategyteam. Akka 3.6.0. HTTP port 9788.

Components to wire (exactly):
- 4 AutonomousAgents, each with a definition() declaring instructions and
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)):
  - CampaignLeadAgent (supervisor): assembles a CampaignPlan{summary} from the
    recorded findings/strategy/content, and in a second task produces a
    PlanEval{score,notes} scoring the plan against the brief.
  - MarketResearchAgent (worker): returns MarketFindings{claims,sources};
    calls the in-process web search at GET /api/search?q=... via an HTTP client
    and grounds every claim in a returned result.
  - StrategyAgent (worker): returns StrategyDraft{positioning,channels,tactics}.
  - ContentAgent (worker): returns ContentArtifacts{taglines,posts}.
- 1 Workflow StrategyWorkflow with steps researchStep -> strategyStep ->
  contentStep -> assembleStep -> evalStep. Each agent step calls
  forAutonomousAgent(Agent.class, scoped-id).runSingleTask(...) then
  forTask(taskId).result(...), and writes the result onto CampaignEntity.
  assembleStep runs the G1 claim-grounding guardrail: if a plan claim is not
  present in the recorded findings, fail the step and retry (max 2). evalStep
  records the self-eval. Override settings() with stepTimeout(60s) on each
  agent-calling step and defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity CampaignEntity holding a Campaign record (id, brief
  Optional<String>, CampaignStatus enum, and Optional lifecycle fields per
  reference/data-model.md). Events: BriefReceived, ResearchRecorded,
  StrategyRecorded, ContentRecorded, PlanAssembled, PlanEvaluated,
  CampaignFailed. Commands: receiveBrief, recordResearch, recordStrategy,
  recordContent, assemblePlan, recordEvaluation, fail, getCampaign.
  emptyState() returns Campaign.initial("") with no commandContext() reference.
- 1 EventSourcedEntity BriefQueue, single command enqueueBrief(brief) emitting
  BriefEnqueued.
- 1 View CampaignsView, row type Campaign, table updater consuming
  CampaignEntity events. ONE query getAllCampaigns SELECT * AS campaigns FROM
  campaigns_view. No WHERE status filter (Akka cannot auto-index enum columns);
  filter client-side in callers. Add a streamAllCampaigns query for SSE.
- 1 Consumer BriefConsumer subscribed to BriefQueue events; on each event
  starts a StrategyWorkflow with a fresh UUID.
- 1 TimedAction BriefSimulator (every 45s, reads next line from
  src/main/resources/sample-events/marketing-briefs.jsonl and calls
  BriefQueue.enqueueBrief).
- 3 HttpEndpoints: CampaignEndpoint at /api (briefs, campaigns list filtered
  client-side from getAllCampaigns, single campaign, SSE stream, three
  /api/metadata/* endpoints serving files from src/main/resources/metadata/);
  WebSearchEndpoint at /api/search returning canned results from
  src/main/resources/sample-docs/search-index.json matched by query terms;
  AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.
- 1 service-setup Bootstrap scheduling BriefSimulator on startup.

Companion files:
- Tasks declared in agent companion classes: CampaignLeadTasks (ASSEMBLE ->
  CampaignPlan, EVALUATE -> PlanEval), MarketResearchTasks (RESEARCH ->
  MarketFindings), StrategyTasks (STRATEGIZE -> StrategyDraft), ContentTasks
  (DRAFT_CONTENT -> ContentArtifacts), each Task.name(...).description(...)
  .resultConformsTo(R.class). An AutonomousAgent without its tasks file is a
  compile error (Lesson 7).
- Result records: MarketFindings(List<String> claims, List<String> sources),
  StrategyDraft(String positioning, List<String> channels, List<String>
  tactics), ContentArtifacts(List<String> taglines, List<String> posts),
  CampaignPlan(String summary), PlanEval(int score, String notes).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9788 and agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/sample-events/marketing-briefs.jsonl with 8 canned briefs.
- src/main/resources/sample-docs/search-index.json with grounding results.
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md}
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at project root with controls G1 and E1 and matching
  simplified_view. No regulation_anchors.
- risk-survey.yaml at project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking deployer-specific fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at project root per the exemplar structure. No
  governance-mechanisms section, no configuration section (Lesson 20).
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm). Inline CSS + JS; runtime CDN imports for
  markdown and YAML are acceptable. Five tabs (Overview, Architecture, Risk
  Survey, Eval Matrix, App UI). Match the governance.html visual style
  (dark / yellow accent / Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (CampaignLeadAgent -> CampaignPlan + PlanEval, MarketResearchAgent ->
  MarketFindings, StrategyAgent -> StrategyDraft, ContentAgent ->
  ContentArtifacts; write src/main/resources/mock-responses/{campaign-lead,
  market-research,strategy,content}.json with 4-6 entries each). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, or secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; it never echoes captured key
  material.

Mock LLM provider — per-agent response shapes when the user picks option (a):
- CampaignLeadAgent: CampaignPlan{summary} stitched from recorded artifacts;
  PlanEval{score 60-95, notes referencing the brief}.
- MarketResearchAgent: MarketFindings{3-5 claims drawn from the canned search
  results, matching sources list}.
- StrategyAgent: StrategyDraft{one positioning line, 3-4 channels, 3-5 tactics}.
- ContentAgent: ContentArtifacts{3 taglines, 3 short posts}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; the extends
  clause matches this spec verbatim.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout
  (60s); the default 5s timeout is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable lifecycle field on the Campaign
  record (it is the View row type).
- Lesson 7: each AutonomousAgent ships its companion Tasks class.
- Lesson 8: verify model names are current before writing them.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9788 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: descriptive integration label ("Runs out of the box"), never T1.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: mermaid state-label CSS overrides + theme variables on the
  Architecture tab (white state labels, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, never NodeList
  index; delete removed panels, never display:none zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and the PLAN.md.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9788/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model-provider key reference (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
