# SPEC — startup-advisor-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Startup Advisor (Multi-Agent).
**One-line pitch:** Submit a startup profile; a supervisor delegates market, GTM, content, and roadmap work to four specialist agents in parallel, then synthesises one advisory report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to four AutonomousAgents in parallel, gathers their results, and asks a fifth AutonomousAgent to synthesise a unified advisory report. The blueprint also demonstrates an **output guardrail** that vets the advisory report for unsupported claims before it is returned, and **eval-event** governance that samples the supervisor's synthesis decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a startup profile via the form.

1. The system creates an `AdvisorySession` record in `PLANNING` and starts an `AdvisoryWorkflow`.
2. The AdvisorSupervisor decomposes the profile into four parallel work items: a market-research query, a GTM strategy brief, a content-plan brief, and a roadmap brief.
3. The workflow forks: all four agents run concurrently. Each returns a typed payload.
4. The AdvisorSupervisor merges the four payloads into an `AdvisoryReport { executiveSummary, marketLandscape, gtmStrategy, contentPlan, roadmap, guardrailVerdict }`.
5. An output guardrail vets the advisory report; if it fails, the session moves to `BLOCKED`. Otherwise, the session moves to `SYNTHESISED`.
6. If any worker times out after 60 seconds, the workflow short-circuits: it asks the AdvisorSupervisor to synthesise from whichever outputs returned, and the session enters `DEGRADED`.

A `ProfileSimulator` (TimedAction) drips a sample startup profile every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AdvisorSupervisor` | `AutonomousAgent` | Decomposes the startup profile, synthesises the merged report, runs the output guardrail. | `AdvisoryWorkflow` | returns typed result to workflow |
| `MarketResearcher` | `AutonomousAgent` | Maps the competitive landscape and target customer segments. | `AdvisoryWorkflow` | — |
| `GtmStrategist` | `AutonomousAgent` | Drafts go-to-market channels, pricing signals, and launch sequencing. | `AdvisoryWorkflow` | — |
| `ContentPlanner` | `AutonomousAgent` | Outlines a content and thought-leadership calendar. | `AdvisoryWorkflow` | — |
| `RoadmapAdvisor` | `AutonomousAgent` | Proposes a phased product roadmap with milestones. | `AdvisoryWorkflow` | — |
| `AdvisoryWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, the guardrail. | `AdvisoryEndpoint`, `ProfileConsumer` | `AdvisorySessionEntity` |
| `AdvisorySessionEntity` | `EventSourcedEntity` | Holds the session's lifecycle (planning → in-progress → synthesised / degraded / blocked). | `AdvisoryWorkflow` | `AdvisoryView` |
| `ProfileQueue` | `EventSourcedEntity` | Logs each submitted profile for replay/audit. | `AdvisoryEndpoint`, `ProfileSimulator` | `ProfileConsumer` |
| `AdvisoryView` | `View` | List-of-sessions read model. | `AdvisorySessionEntity` events | `AdvisoryEndpoint` |
| `ProfileConsumer` | `Consumer` | Listens to `ProfileQueue` events and starts a workflow per submission. | `ProfileQueue` events | `AdvisoryWorkflow` |
| `ProfileSimulator` | `TimedAction` | Drips a sample startup profile every 60 s. | scheduler | `ProfileQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesised session every 5 minutes for eval scoring; emits an `EvalScored` event. | scheduler | `AdvisorySessionEntity` |
| `AdvisoryEndpoint` | `HttpEndpoint` | `/api/advisory/*` — submit, get, list, SSE. | — | `AdvisoryView`, `ProfileQueue`, `AdvisorySessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record StartupProfile(String companyName, String sector, String stage,
                      String problemStatement, String requestedBy) {}

record MarketLandscape(List<Competitor> competitors, List<String> targetSegments,
                       String marketSizeSummary, Instant researchedAt) {}
record Competitor(String name, String positioning, String differentiator) {}

record GtmStrategy(List<String> channels, String pricingSignal,
                   List<String> launchSequence, Instant strategyAt) {}

record ContentPlan(List<ContentPillar> pillars, List<String> formats,
                   String cadenceRecommendation, Instant plannedAt) {}
record ContentPillar(String theme, String rationale) {}

record ProductRoadmap(List<RoadmapPhase> phases, String keyRisk, Instant roadmappedAt) {}
record RoadmapPhase(String name, String goal, List<String> milestones) {}

record WorkItems(String marketQuery, String gtmBrief, String contentBrief, String roadmapBrief) {}

record AdvisoryReport(String executiveSummary, MarketLandscape marketLandscape,
                      GtmStrategy gtmStrategy, ContentPlan contentPlan,
                      ProductRoadmap roadmap, String guardrailVerdict,
                      Instant synthesisedAt) {}

record AdvisorySession(
    String sessionId,
    String companyName,
    String sector,
    SessionStatus status,
    Optional<MarketLandscape> marketLandscape,
    Optional<GtmStrategy> gtmStrategy,
    Optional<ContentPlan> contentPlan,
    Optional<ProductRoadmap> roadmap,
    Optional<AdvisoryReport> report,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

### Events (on `AdvisorySessionEntity`)

`SessionCreated`, `MarketLandscapeAttached`, `GtmStrategyAttached`, `ContentPlanAttached`,
`RoadmapAttached`, `ReportSynthesised`, `SessionDegraded`, `SessionBlocked`, `EvalScored`.

### Events (on `ProfileQueue`)

`ProfileSubmitted { sessionId, companyName, sector, stage, problemStatement, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/advisory` — body `{ companyName, sector, stage, problemStatement }` → `{ sessionId }`. Starts a workflow.
- `GET /api/advisory` — list all sessions. Optional `?status=PLANNING|IN_PROGRESS|SYNTHESISED|DEGRADED|BLOCKED`.
- `GET /api/advisory/{id}` — one session.
- `GET /api/advisory/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Startup Advisor (Multi-Agent)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a startup profile, live list of sessions with status pills, expand-row to see market landscape + GTM strategy + content plan + roadmap + synthesised summary + eval score.

Browser title: `<title>Akka Sample: Startup Advisor (Multi-Agent)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `AdvisorSupervisor`): vets the advisory report for unsupported commercial claims, fabricated competitor data, and explicit sales manipulation. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one synthesised session every 5 minutes and emits an `EvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `AdvisorSupervisor` → `prompts/advisor-supervisor.md`. Decomposes the profile into work items; later synthesises specialist outputs into the final advisory report.
- `MarketResearcher` → `prompts/market-researcher.md`. Maps competitive landscape; returns `MarketLandscape`.
- `GtmStrategist` → `prompts/gtm-strategist.md`. Drafts channels, pricing signals, and launch sequence; returns `GtmStrategy`.
- `ContentPlanner` → `prompts/content-planner.md`. Outlines content pillars and formats; returns `ContentPlan`.
- `RoadmapAdvisor` → `prompts/roadmap-advisor.md`. Proposes phased roadmap with milestones; returns `ProductRoadmap`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a startup profile; session progresses PLANNING → IN_PROGRESS → SYNTHESISED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `MarketResearcher` timeout to 1 s); session enters DEGRADED with whichever partial outputs came back.
3. **J3** — Inject a guardrail failure (AdvisorSupervisor returns a report with fabricated competitor claims); session enters BLOCKED.
4. **J4** — Wait after a successful synthesis; the session's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named startup-advisor-team demonstrating the
delegation-supervisor-workers × sales-marketing cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-sales-marketing-startup-advisor-team.
Java package io.akka.samples.startupadvisormultiagent. Akka 3.6.0.
HTTP port 9164.

Components to wire (exactly):
- 5 AutonomousAgents:
  * AdvisorSupervisor — definition() with
    capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/advisor-supervisor.md. Returns
    WorkItems{marketQuery, gtmBrief, contentBrief, roadmapBrief} for DECOMPOSE
    and AdvisoryReport{executiveSummary, marketLandscape, gtmStrategy,
    contentPlan, roadmap, guardrailVerdict, synthesisedAt} for SYNTHESISE.
  * MarketResearcher — capability(TaskAcceptance.of(RESEARCH_MARKET).maxIterationsPerTask(3)).
    System prompt from prompts/market-researcher.md. Returns
    MarketLandscape{competitors: List<Competitor{name, positioning, differentiator}>,
    targetSegments: List<String>, marketSizeSummary, researchedAt}.
  * GtmStrategist — capability(TaskAcceptance.of(PLAN_GTM).maxIterationsPerTask(3)).
    System prompt from prompts/gtm-strategist.md. Returns
    GtmStrategy{channels: List<String>, pricingSignal, launchSequence: List<String>,
    strategyAt}.
  * ContentPlanner — capability(TaskAcceptance.of(PLAN_CONTENT).maxIterationsPerTask(2)).
    System prompt from prompts/content-planner.md. Returns
    ContentPlan{pillars: List<ContentPillar{theme, rationale}>, formats: List<String>,
    cadenceRecommendation, plannedAt}.
  * RoadmapAdvisor — capability(TaskAcceptance.of(PLAN_ROADMAP).maxIterationsPerTask(2)).
    System prompt from prompts/roadmap-advisor.md. Returns
    ProductRoadmap{phases: List<RoadmapPhase{name, goal, milestones: List<String>}>,
    keyRisk, roadmappedAt}.

- 1 Workflow AdvisoryWorkflow with steps:
  decomposeStep -> [parallel] marketStep, gtmStep, contentStep, roadmapStep
  -> joinStep -> synthesiseStep -> guardrailStep -> emitStep.
  decomposeStep calls forAutonomousAgent(AdvisorSupervisor.class, DECOMPOSE).
  marketStep, gtmStep, contentStep, roadmapStep run in parallel
  (CompletionStage allOf zip); each governed by
  WorkflowSettings.builder().stepTimeout(AdvisoryWorkflow::marketStep, ofSeconds(60))
  and equivalent 60s timeouts for gtmStep, contentStep, roadmapStep.
  On any timeout, transition to a degradeStep that calls synthesiseStep with
  whichever specialist outputs returned, then ends with SessionDegraded.
  synthesiseStep calls forAutonomousAgent(AdvisorSupervisor.class, SYNTHESISE)
  with the merged inputs; give synthesiseStep a 120s stepTimeout.
  guardrailStep runs the deterministic vetter + LLM judge on the advisory content;
  on failure, end with SessionBlocked. WorkflowSettings is nested inside
  Workflow — no import.

- 1 EventSourcedEntity AdvisorySessionEntity holding state AdvisorySession{sessionId,
  companyName, sector, SessionStatus, Optional<MarketLandscape> marketLandscape,
  Optional<GtmStrategy> gtmStrategy, Optional<ContentPlan> contentPlan,
  Optional<ProductRoadmap> roadmap, Optional<AdvisoryReport> report,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  SessionStatus enum: PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED.
  Events: SessionCreated, MarketLandscapeAttached, GtmStrategyAttached,
  ContentPlanAttached, RoadmapAttached, ReportSynthesised, SessionDegraded,
  SessionBlocked, EvalScored.
  Commands: createSession, attachMarketLandscape, attachGtmStrategy,
  attachContentPlan, attachRoadmap, synthesise, degrade, block, recordEval, getSession.
  emptyState() returns AdvisorySession.initial("", "", "") with no commandContext() reference.

- 1 EventSourcedEntity ProfileQueue with command enqueueProfile(companyName, sector, stage,
  problemStatement, requestedBy) emitting ProfileSubmitted{sessionId, companyName, sector,
  stage, problemStatement, requestedBy, submittedAt}.

- 1 View AdvisoryView with row type AdvisorySessionRow (mirrors AdvisorySession minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  AdvisorySessionEntity events. ONE query getAllSessions SELECT * AS sessions FROM advisory_view.
  No WHERE status filter — caller filters client-side.

- 1 Consumer ProfileConsumer subscribed to ProfileQueue events; on ProfileSubmitted
  starts an AdvisoryWorkflow with the sessionId as the workflow id.

- 2 TimedActions:
  * ProfileSimulator — every 60s, reads next line from
    src/main/resources/sample-events/startup-profiles.jsonl and calls
    ProfileQueue.enqueueProfile.
  * EvalSampler — every 5 minutes, queries AdvisoryView.getAllSessions, picks the oldest
    SYNTHESISED session without an evalScore, runs a 1-5 rubric judge over the advisory
    content, then calls AdvisorySessionEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * AdvisoryEndpoint at /api with POST /advisory, GET /advisory, GET /advisory/{id},
    GET /advisory/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- AdvisorTasks.java declaring five Task<R> constants: DECOMPOSE (WorkItems),
  RESEARCH_MARKET (MarketLandscape), PLAN_GTM (GtmStrategy),
  PLAN_CONTENT (ContentPlan), PLAN_ROADMAP (ProductRoadmap),
  SYNTHESISE (AdvisoryReport).
- Domain records WorkItems, Competitor, MarketLandscape, GtmStrategy, ContentPillar,
  ContentPlan, RoadmapPhase, ProductRoadmap, AdvisoryReport, StartupProfile.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9164 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/startup-profiles.jsonl with 8 canned startup
  profile lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view
  list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  startup-advisory, decisions.authority_level = recommend-only,
  data.data_classes.pii = false, capabilities.* = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/advisor-supervisor.md, prompts/market-researcher.md, prompts/gtm-strategist.md,
  prompts/content-planner.md, prompts/roadmap-advisor.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Startup Advisor (Multi-Agent)",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams
  + click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Startup Advisor (Multi-Agent)</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (advisor-supervisor.json,
  market-researcher.json, gtm-strategist.json, content-planner.json,
  roadmap-advisor.json), picks one entry pseudo-randomly per call, and
  deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    advisor-supervisor.json — list of either WorkItems or AdvisoryReport objects.
      4–6 WorkItems entries (marketQuery + gtmBrief + contentBrief + roadmapBrief
      tuples) and 4–6 AdvisoryReport entries (each with a 100–150 word
      executiveSummary, a populated MarketLandscape, GtmStrategy, ContentPlan,
      ProductRoadmap, guardrailVerdict = "ok").
    market-researcher.json — 4–6 MarketLandscape entries, each with 3–5 competitors
      (name + positioning + differentiator), 2–4 targetSegments, a marketSizeSummary
      of one to two sentences.
    gtm-strategist.json — 4–6 GtmStrategy entries, each with 3–5 channels,
      a pricingSignal sentence, and 3–5 launchSequence steps.
    content-planner.json — 4–6 ContentPlan entries, each with 3–5 ContentPillar
      items (theme + rationale), 2–4 formats, and a cadenceRecommendation.
    roadmap-advisor.json — 4–6 ProductRoadmap entries, each with 3 RoadmapPhase
      items (name + goal + 2–4 milestones) and a keyRisk sentence.
- A MockModelProvider.seedFor(sessionId) helper makes the selection
  deterministic per session id so the same session produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  120s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion AdvisorTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9164 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black
  and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
