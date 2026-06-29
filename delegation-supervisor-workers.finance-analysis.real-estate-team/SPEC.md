# SPEC — real-estate-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Real Estate Investment Multi-Agent.
**One-line pitch:** Submit a property deal; a coordinator delegates market research and financial modeling to two specialists in parallel, then synthesises a single investment recommendation with regulated output gating.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise an investment recommendation. The blueprint also demonstrates a **sector sanitizer** that screens property submissions for regulatory exclusions before any agent runs, and an **output guardrail** that vets the recommendation before it is returned. A periodic eval sampler scores the coordinator's synthesis decision for quality.

## 3. User-facing flows

The user opens the App UI tab and submits a property deal via the form.

1. The system applies a sector sanitizer — if the property type is excluded (e.g., cannabis facility, unlicensed rental), the deal enters `REJECTED` immediately and no agents run.
2. If the sanitizer passes, the system creates a `DealRecord` in `SCOPING` and starts an `EvaluationWorkflow`.
3. The Coordinator decomposes the deal into two parallel work items: a market research query for the MarketSpecialist, and a financial modeling question for the FinanceSpecialist.
4. The workflow forks: both agents run concurrently. Each returns a typed payload.
5. The Coordinator merges the two payloads into an `InvestmentRecommendation { summary, marketReport, financialModel, recommendationVerdict, guardrailVerdict }`.
6. An output guardrail vets the recommendation; if it fails, the deal moves to `BLOCKED`. Otherwise, the deal moves to `RECOMMENDED`.
7. If either worker times out after 60 seconds, the workflow short-circuits: the Coordinator synthesises from whichever side returned, and the deal enters `DEGRADED`.

A `DealSimulator` (TimedAction) drips a sample property scenario every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DealCoordinator` | `AutonomousAgent` | Decomposes the deal into parallel work items; synthesises the merged recommendation; runs the output guardrail. | `EvaluationWorkflow` | returns typed result to workflow |
| `MarketSpecialist` | `AutonomousAgent` | Gathers comparable sales, neighborhood trends, and pricing data. Seeded market tools return canned results. | `EvaluationWorkflow` | — |
| `FinanceSpecialist` | `AutonomousAgent` | Models cap rate, NOI, cash-on-cash return, and DSCR. | `EvaluationWorkflow` | — |
| `EvaluationWorkflow` | `Workflow` | Coordinates sector sanitizer check, parallel fan-out, synthesis, and output guardrail. | `DealEndpoint`, `DealRequestConsumer` | `DealEntity` |
| `DealEntity` | `EventSourcedEntity` | Holds the deal's lifecycle (scoping → evaluating → recommended / degraded / blocked / rejected). | `EvaluationWorkflow` | `DealView` |
| `DealQueue` | `EventSourcedEntity` | Logs each submitted deal for replay/audit. | `DealEndpoint`, `DealSimulator` | `DealRequestConsumer` |
| `DealView` | `View` | List-of-deals read model. | `DealEntity` events | `DealEndpoint` |
| `DealRequestConsumer` | `Consumer` | Listens to `DealQueue` events and starts a workflow per submission. | `DealQueue` events | `EvaluationWorkflow` |
| `DealSimulator` | `TimedAction` | Drips a sample property deal every 60 s. | scheduler | `DealQueue` |
| `RecommendationEvalSampler` | `TimedAction` | Samples one recommended deal every 5 minutes for eval scoring; emits a `RecommendationScored` event. | scheduler | `DealEntity` |
| `DealEndpoint` | `HttpEndpoint` | `/api/deals/*` — submit, get, list, SSE. | — | `DealView`, `DealQueue`, `DealEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record DealSubmission(String address, String propertyType, double askingPriceDollars,
                      String submittedBy) {}

record MarketReport(List<Comparable> comparables, String neighborhoodTrend,
                    double estimatedMarketValueDollars, Instant gatheredAt) {}
record Comparable(String address, double salePriceDollars, String saleDate, double sqft) {}

record FinancialModel(double grossRentalIncome, double operatingExpenses,
                      double netOperatingIncome, double capRate,
                      double cashOnCashReturn, double debtServiceCoverageRatio,
                      Instant modelledAt) {}

record EvaluationPlan(String marketQuery, String financialQuestion) {}

record InvestmentRecommendation(String summary, MarketReport marketReport,
                                FinancialModel financialModel,
                                String recommendationVerdict,
                                String guardrailVerdict,
                                Instant synthesisedAt) {}

record DealRecord(
    String dealId,
    String address,
    String propertyType,
    double askingPriceDollars,
    DealStatus status,
    Optional<MarketReport> marketReport,
    Optional<FinancialModel> financialModel,
    Optional<InvestmentRecommendation> recommendation,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DealStatus { SCOPING, EVALUATING, RECOMMENDED, DEGRADED, BLOCKED, REJECTED }
```

### Events (on `DealEntity`)

`DealCreated`, `DealRejected`, `MarketReportAttached`, `FinancialModelAttached`, `DealRecommended`, `DealDegraded`, `DealBlocked`, `RecommendationScored`.

### Events (on `DealQueue`)

`DealSubmitted { dealId, address, propertyType, askingPriceDollars, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/deals` — body `{ address, propertyType, askingPriceDollars }` → `{ dealId }`. Starts a workflow.
- `GET /api/deals` — list all deals. Optional `?status=SCOPING|EVALUATING|RECOMMENDED|DEGRADED|BLOCKED|REJECTED`.
- `GET /api/deals/{id}` — one deal.
- `GET /api/deals/sse` — server-sent events stream of every deal change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Real Estate Investment Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a property deal (address, property type, asking price), live list of deals with status pills, expand-row to see market report + financial model + recommendation + eval score.

Browser title: `<title>Akka Sample: Real Estate Investment Multi-Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — sector sanitizer** (`sector` flavor): screens `propertyType` against a regulated-exclusion list before any agent task runs. Blocking. Failure → `REJECTED`.
- **G1 — output guardrail** (`before-agent-response` on `DealCoordinator`): vets the synthesised recommendation for fabricated financial metrics and regulatory non-compliance. Blocking. Failure → `BLOCKED`.

## 9. Agent prompts

- `DealCoordinator` → `prompts/deal-coordinator.md`. Decomposes the deal into parallel work items; later synthesises a final investment recommendation.
- `MarketSpecialist` → `prompts/market-specialist.md`. Gathers market comparables and neighborhood trend data; returns `MarketReport`.
- `FinanceSpecialist` → `prompts/finance-specialist.md`. Models the deal's financial returns; returns `FinancialModel`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a valid property deal; deal progresses SCOPING → EVALUATING → RECOMMENDED within 60 s; UI reflects each transition via SSE.
2. **J2** — Submit a property type on the exclusion list; deal enters REJECTED before any agent runs.
3. **J3** — Inject a worker timeout (set `MarketSpecialist` timeout to 1 s); deal enters DEGRADED with whichever partial output came back.
4. **J4** — Inject a guardrail failure (Coordinator returns a recommendation with a fabricated metric); deal enters BLOCKED.
5. **J5** — Wait after a successful recommendation; the deal's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named real-estate-team demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-real-estate-team.
Java package io.akka.samples.realestateinvestmentmultiagent. Akka 3.6.0. HTTP port 9581.

Components to wire (exactly):
- 3 AutonomousAgents:
  * DealCoordinator — definition() with capability(TaskAcceptance.of(SCOPE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/deal-coordinator.md. Returns EvaluationPlan{marketQuery, financialQuestion} for
    SCOPE and InvestmentRecommendation{summary, marketReport, financialModel,
    recommendationVerdict, guardrailVerdict, synthesisedAt} for SYNTHESISE.
  * MarketSpecialist — capability(TaskAcceptance.of(RESEARCH_MARKET).maxIterationsPerTask(3)).
    System prompt from prompts/market-specialist.md. Returns
    MarketReport{comparables: List<Comparable{address, salePriceDollars, saleDate, sqft}>,
    neighborhoodTrend, estimatedMarketValueDollars, gatheredAt}.
  * FinanceSpecialist — capability(TaskAcceptance.of(MODEL_FINANCE).maxIterationsPerTask(2)).
    System prompt from prompts/finance-specialist.md. Returns
    FinancialModel{grossRentalIncome, operatingExpenses, netOperatingIncome, capRate,
    cashOnCashReturn, debtServiceCoverageRatio, modelledAt}.

- 1 Workflow EvaluationWorkflow with steps:
  sanitizerStep -> scopeStep -> [parallel] marketStep, financeStep -> joinStep ->
  synthesiseStep -> guardrailStep -> emitStep.
  sanitizerStep checks DealSubmission.propertyType against a hardcoded exclusion set
  (cannabis_facility, unlicensed_short_term_rental, gambling_establishment); if matched,
  ends immediately with DealRejected — no agent call.
  scopeStep calls forAutonomousAgent(DealCoordinator.class, SCOPE).
  marketStep and financeStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(EvaluationWorkflow::marketStep, ofSeconds(60)) and
  stepTimeout(EvaluationWorkflow::financeStep, ofSeconds(60)). On either timeout, transition to
  a degradeStep that calls synthesiseStep with whichever side returned, then ends with
  DealDegraded.
  synthesiseStep calls forAutonomousAgent(DealCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep runs the deterministic vetter +
  LLM judge on the synthesised recommendation; on failure, end with DealBlocked. WorkflowSettings
  is nested inside Workflow — no import.

- 1 EventSourcedEntity DealEntity holding state DealRecord{dealId, address, propertyType,
  askingPriceDollars, DealStatus, Optional<MarketReport> marketReport,
  Optional<FinancialModel> financialModel, Optional<InvestmentRecommendation> recommendation,
  Optional<String> failureReason, Optional<Integer> evalScore, Optional<String> evalRationale,
  Instant createdAt, Optional<Instant> finishedAt}. DealStatus enum: SCOPING, EVALUATING,
  RECOMMENDED, DEGRADED, BLOCKED, REJECTED. Events: DealCreated, DealRejected,
  MarketReportAttached, FinancialModelAttached, DealRecommended, DealDegraded, DealBlocked,
  RecommendationScored. Commands: createDeal, rejectDeal, attachMarketReport,
  attachFinancialModel, recommend, degrade, block, recordScore, getDeal.
  emptyState() returns DealRecord.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity DealQueue with command submitDeal(address, propertyType,
  askingPriceDollars, submittedBy) emitting DealSubmitted{dealId, address, propertyType,
  askingPriceDollars, submittedBy, submittedAt}.

- 1 View DealView with row type DealRow (mirrors DealRecord minus heavy nested payloads;
  every nullable field is Optional<T>). Table updater consumes DealEntity events. ONE query
  getAllDeals SELECT * AS deals FROM deal_view. No WHERE status filter (Akka cannot
  auto-index enum columns) — caller filters client-side.

- 1 Consumer DealRequestConsumer subscribed to DealQueue events; on DealSubmitted starts an
  EvaluationWorkflow with the dealId as the workflow id.

- 2 TimedActions:
  * DealSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-deals.jsonl and calls DealQueue.submitDeal.
  * RecommendationEvalSampler — every 5 minutes, queries DealView.getAllDeals, picks the oldest
    RECOMMENDED deal without an evalScore, runs a 1–5 rubric judge over the synthesised
    recommendation, then calls DealEntity.recordScore(score, rationale).

- 2 HttpEndpoints:
  * DealEndpoint at /api with POST /deals, GET /deals, GET /deals/{id},
    GET /deals/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DealTasks.java declaring four Task<R> constants: SCOPE (EvaluationPlan), RESEARCH_MARKET
  (MarketReport), MODEL_FINANCE (FinancialModel), SYNTHESISE (InvestmentRecommendation).
- Domain records EvaluationPlan, Comparable, MarketReport, FinancialModel,
  InvestmentRecommendation, DealSubmission.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9581 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-deals.jsonl with 8 canned property deal lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (S1 sector sanitizer,
  G1 output guardrail before-agent-response) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  investment-analysis, decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false except content_generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/deal-coordinator.md, prompts/market-specialist.md, prompts/finance-specialist.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Real Estate Investment Multi-Agent",
  one-line pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Real Estate Investment Multi-Agent</title>.
  No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (deal-coordinator.json,
  market-specialist.json, finance-specialist.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    deal-coordinator.json — list of either EvaluationPlan or InvestmentRecommendation objects.
      4–6 EvaluationPlan entries (marketQuery + financialQuestion pairs describing real
      property scenarios) and 4–6 InvestmentRecommendation entries (each with an 80–150 word
      summary, a MarketReport stub, a FinancialModel stub, recommendationVerdict = "proceed"
      or "pass", guardrailVerdict = "ok").
    market-specialist.json — 4–6 MarketReport entries, each with 3–6 Comparable sales
      (plausible US addresses, prices $200k–$2M, dates within 12 months), a neighborhoodTrend
      string ("appreciating", "stable", "softening"), and an estimatedMarketValueDollars value.
    finance-specialist.json — 4–6 FinancialModel entries with realistic cap rates (4–8%),
      cash-on-cash returns (5–12%), DSCR values (1.0–1.8), and positive NOI figures.
- A MockModelProvider.seedFor(dealId) helper makes the selection deterministic per deal id
  so the same deal produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion DealTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9581 in application.conf.
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
- The sanitizerStep runs BEFORE any agent task — it is a pure Java check, not an agent call.
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
