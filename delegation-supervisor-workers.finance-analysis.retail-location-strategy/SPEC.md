# SPEC — retail-ai-location-strategy

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Retail AI Location Strategy.
**One-line pitch:** Submit a candidate store address; a coordinator delegates market scoring to a Market Analyst and demographic scoring to a Demographics Analyst in parallel, then synthesises one ranked location recommendation.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise a ranked recommendation. The blueprint also demonstrates an **output guardrail** that vets the recommendation before it is returned to the requester, and **eval-event** governance that samples the coordinator's synthesis decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a candidate site address via the form.

1. The system creates a `CandidateSite` record in `SCORING` and starts a `LocationWorkflow`.
2. The Coordinator decomposes the candidate site into two parallel work items: a market-conditions query for the Market Analyst, and a demographic-fit question for the Demographics Analyst.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `SiteRecommendation { summary, marketAssessment, demographicAssessment, score, verdict, guardrailVerdict }`.
5. An output guardrail vets the recommendation; if it fails, the evaluation moves to `BLOCKED`. Otherwise, verdict determines whether the final state is `RECOMMENDED` or `NOT_RECOMMENDED`.
6. If either analyst times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to synthesise from whichever side returned, and the evaluation enters `DEGRADED`.

A `SiteSimulator` (TimedAction) drips a sample candidate site every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LocationCoordinator` | `AutonomousAgent` | Decomposes the candidate site into scoring tasks, synthesises the merged recommendation, runs the output guardrail. | `LocationWorkflow` | returns typed result to workflow |
| `MarketAnalyst` | `AutonomousAgent` | Evaluates trade-area market conditions for the candidate site. Seeded scoring tool returns canned market data. | `LocationWorkflow` | — |
| `DemographicsAnalyst` | `AutonomousAgent` | Assesses population density and consumer-profile fit for the candidate site. | `LocationWorkflow` | — |
| `LocationWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, the guardrail. | `LocationEndpoint`, `SiteRequestConsumer` | `CandidateSiteEntity` |
| `CandidateSiteEntity` | `EventSourcedEntity` | Holds the site evaluation lifecycle (scoring → in-progress → recommended / not-recommended / degraded / blocked). | `LocationWorkflow` | `LocationView` |
| `SiteQueue` | `EventSourcedEntity` | Logs each submitted candidate site for replay/audit. | `LocationEndpoint`, `SiteSimulator` | `SiteRequestConsumer` |
| `LocationView` | `View` | List-of-evaluations read model. | `CandidateSiteEntity` events | `LocationEndpoint` |
| `SiteRequestConsumer` | `Consumer` | Listens to `SiteQueue` events and starts a workflow per submission. | `SiteQueue` events | `LocationWorkflow` |
| `SiteSimulator` | `TimedAction` | Drips a sample candidate site every 60 s. | scheduler | `SiteQueue` |
| `EvalSampler` | `TimedAction` | Samples one finished recommendation every 5 minutes for eval scoring; emits an `EvalScored` event. | scheduler | `CandidateSiteEntity` |
| `LocationEndpoint` | `HttpEndpoint` | `/api/locations/*` — submit, get, list, SSE. | — | `LocationView`, `SiteQueue`, `CandidateSiteEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record SiteSubmission(String address, String city, String region, String requestedBy) {}

record MarketAssessment(double trafficScore, double competitorDensity,
                        String tradeAreaClassification, List<String> keyInsights,
                        Instant assessedAt) {}

record DemographicAssessment(double populationScore, double incomeAlignmentScore,
                             String consumerProfile, List<String> keyInsights,
                             Instant assessedAt) {}

record ScoringPlan(String marketQuery, String demographicQuestion) {}

record SiteRecommendation(String summary, MarketAssessment marketAssessment,
                          DemographicAssessment demographicAssessment,
                          double score, LocationVerdict verdict,
                          String guardrailVerdict, Instant synthesisedAt) {}

record CandidateSite(
    String siteId,
    String address,
    String city,
    String region,
    SiteStatus status,
    Optional<MarketAssessment> marketAssessment,
    Optional<DemographicAssessment> demographicAssessment,
    Optional<SiteRecommendation> recommendation,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SiteStatus { SCORING, IN_PROGRESS, RECOMMENDED, NOT_RECOMMENDED, DEGRADED, BLOCKED }
enum LocationVerdict { RECOMMENDED, NOT_RECOMMENDED }
```

### Events (on `CandidateSiteEntity`)

`SiteCreated`, `MarketAssessmentAttached`, `DemographicAssessmentAttached`, `SiteRecommended`, `SiteNotRecommended`, `SiteDegraded`, `SiteBlocked`, `EvalScored`.

### Events (on `SiteQueue`)

`SiteSubmitted { siteId, address, city, region, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/locations` — body `{ address, city, region }` → `{ siteId }`. Starts a workflow.
- `GET /api/locations` — list all evaluations. Optional `?status=SCORING|IN_PROGRESS|RECOMMENDED|NOT_RECOMMENDED|DEGRADED|BLOCKED`.
- `GET /api/locations/{id}` — one evaluation.
- `GET /api/locations/sse` — server-sent events stream of every evaluation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Retail AI Location Strategy"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a candidate site (address, city, region), live list of evaluations with status pills, expand-row to see market assessment + demographic assessment + recommendation summary + eval score.

Browser title: `<title>Akka Sample: Retail AI Location Strategy</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `LocationCoordinator`): vets the site recommendation for unsupported scoring claims and data-integrity violations. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one finished recommendation every 5 minutes and emits an `EvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `LocationCoordinator` → `prompts/coordinator.md`. Decomposes the candidate site into scoring tasks; later synthesises parallel results into the final recommendation.
- `MarketAnalyst` → `prompts/market-analyst.md`. Evaluates trade-area market conditions; returns `MarketAssessment`.
- `DemographicsAnalyst` → `prompts/demographics-analyst.md`. Assesses population and consumer-profile fit; returns `DemographicAssessment`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a candidate site; evaluation progresses SCORING → IN_PROGRESS → RECOMMENDED (or NOT_RECOMMENDED) within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `MarketAnalyst` timeout to 1 s); evaluation enters DEGRADED with whichever partial assessment came back.
3. **J3** — Inject a guardrail failure (Coordinator returns unsupported score data); evaluation enters BLOCKED.
4. **J4** — Wait after a successful recommendation; the row in the UI shows an eval score.
5. **J5** — Background simulator drips candidate sites every 60 s; App UI is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named retail-ai-location-strategy demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-retail-location-strategy.
Java package io.akka.samples.retailailocationstrategy. Akka 3.6.0. HTTP port 9747.

Components to wire (exactly):
- 3 AutonomousAgents:
  * LocationCoordinator — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/coordinator.md. Returns ScoringPlan{marketQuery, demographicQuestion} for DECOMPOSE
    and SiteRecommendation{summary, marketAssessment, demographicAssessment, score, verdict,
    guardrailVerdict, synthesisedAt} for SYNTHESISE.
  * MarketAnalyst — capability(TaskAcceptance.of(ASSESS_MARKET).maxIterationsPerTask(3)). System
    prompt from prompts/market-analyst.md. Returns MarketAssessment{trafficScore, competitorDensity,
    tradeAreaClassification, keyInsights: List<String>, assessedAt}.
  * DemographicsAnalyst — capability(TaskAcceptance.of(ASSESS_DEMOGRAPHICS).maxIterationsPerTask(2)).
    System prompt from prompts/demographics-analyst.md. Returns DemographicAssessment{populationScore,
    incomeAlignmentScore, consumerProfile, keyInsights: List<String>, assessedAt}.

- 1 Workflow LocationWorkflow with steps:
  decomposeStep -> [parallel] assessMarketStep, assessDemographicsStep -> joinStep -> synthesiseStep
  -> guardrailStep -> emitStep.
  decomposeStep calls forAutonomousAgent(LocationCoordinator.class, DECOMPOSE).
  assessMarketStep and assessDemographicsStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(LocationWorkflow::assessMarketStep, ofSeconds(60)) and
  stepTimeout(LocationWorkflow::assessDemographicsStep, ofSeconds(60)). On either timeout, transition
  to a degradeStep that calls synthesiseStep with whichever side returned, then ends with SiteDegraded.
  synthesiseStep calls forAutonomousAgent(LocationCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep runs the deterministic vetter + LLM
  judge on the recommendation content; on failure, end with SiteBlocked. WorkflowSettings is nested
  inside Workflow — no import.

- 1 EventSourcedEntity CandidateSiteEntity holding state CandidateSite{siteId, address, city, region,
  SiteStatus, Optional<MarketAssessment> marketAssessment, Optional<DemographicAssessment>
  demographicAssessment, Optional<SiteRecommendation> recommendation, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. SiteStatus enum: SCORING, IN_PROGRESS, RECOMMENDED, NOT_RECOMMENDED,
  DEGRADED, BLOCKED. LocationVerdict enum: RECOMMENDED, NOT_RECOMMENDED. Events: SiteCreated,
  MarketAssessmentAttached, DemographicAssessmentAttached, SiteRecommended, SiteNotRecommended,
  SiteDegraded, SiteBlocked, EvalScored. Commands: createSite, attachMarketAssessment,
  attachDemographicAssessment, recommend, notRecommend, degrade, block, recordEval, getSite.
  emptyState() returns CandidateSite.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity SiteQueue with command enqueueSite(address, city, region, requestedBy)
  emitting SiteSubmitted{siteId, address, city, region, requestedBy, submittedAt}.

- 1 View LocationView with row type CandidateSiteRow (mirrors CandidateSite minus heavy nested
  payloads; every nullable field is Optional<T>). Table updater consumes CandidateSiteEntity events.
  ONE query getAllSites SELECT * AS sites FROM location_view. No WHERE status filter (Akka cannot
  auto-index enum columns) — caller filters client-side.

- 1 Consumer SiteRequestConsumer subscribed to SiteQueue events; on SiteSubmitted starts a
  LocationWorkflow with the siteId as the workflow id.

- 2 TimedActions:
  * SiteSimulator — every 60s, reads next line from
    src/main/resources/sample-events/candidate-sites.jsonl and calls SiteQueue.enqueueSite.
  * EvalSampler — every 5 minutes, queries LocationView.getAllSites, picks the oldest
    RECOMMENDED or NOT_RECOMMENDED site without an evalScore, runs a 1-5 rubric judge over the
    recommendation content, then calls CandidateSiteEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * LocationEndpoint at /api with POST /locations, GET /locations, GET /locations/{id},
    GET /locations/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- LocationTasks.java declaring four Task<R> constants: DECOMPOSE (ScoringPlan), ASSESS_MARKET
  (MarketAssessment), ASSESS_DEMOGRAPHICS (DemographicAssessment), SYNTHESISE (SiteRecommendation).
- Domain records ScoringPlan, MarketAssessment, DemographicAssessment, SiteRecommendation,
  SiteSubmission, CandidateSite.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9747 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/candidate-sites.jsonl with 8 canned candidate site lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = retail-location-
  strategy, decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/market-analyst.md, prompts/demographics-analyst.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Retail AI Location Strategy", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Retail AI Location Strategy</title>. No
  subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  market-analyst.json, demographics-analyst.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either ScoringPlan or SiteRecommendation objects.
      4–6 ScoringPlan entries (marketQuery + demographicQuestion pairs) and
      4–6 SiteRecommendation entries (each with an 80–150 word summary, a
      MarketAssessment, a DemographicAssessment, a composite score 0.0–1.0,
      a verdict, and guardrailVerdict = "ok").
    market-analyst.json — 4–6 MarketAssessment entries, each with realistic
      trafficScore (0.0–1.0), competitorDensity (0.0–1.0), a tradeAreaClassification
      (e.g., "suburban-power-strip", "urban-high-street"), and 3–5 keyInsights.
    demographics-analyst.json — 4–6 DemographicAssessment entries, each with
      populationScore (0.0–1.0), incomeAlignmentScore (0.0–1.0), a consumerProfile
      description, and 3–5 keyInsights.
- A MockModelProvider.seedFor(siteId) helper makes the selection deterministic per
  site id so the same site produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s analysts,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion LocationTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9747 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
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
