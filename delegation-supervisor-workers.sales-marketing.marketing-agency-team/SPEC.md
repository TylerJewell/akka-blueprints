# SPEC — marketing-agency-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Marketing Agency Team.
**One-line pitch:** Submit a campaign brief; a director delegates a website launch to one specialist and a positioning strategy to another in parallel, then synthesises a brand-compliant marketing plan.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise a unified marketing plan. The blueprint also demonstrates a **brand-safety guardrail** (`before-agent-response`) that vets the final plan for off-brand copy and policy violations before it is returned.

## 3. User-facing flows

The user opens the App UI tab and submits a campaign brief via the form.

1. The system creates a `CampaignPlan` record in `PLANNING` and starts a `CampaignWorkflow`.
2. The CampaignDirector decomposes the brief into two parallel work items: a website launch scope for the WebsiteLauncher, and a positioning brief for the StrategyAdvisor.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The CampaignDirector merges the two payloads into a `SynthesisedPlan { executiveSummary, launchBrief, strategyFramework, guardrailVerdict }`.
5. A brand-safety guardrail vets the synthesised plan; if it fails, the plan moves to `BLOCKED`. Otherwise, the plan moves to `SYNTHESISED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the CampaignDirector to synthesise from whichever side returned, and the plan enters `DEGRADED`.

A `RequestSimulator` (TimedAction) drips a sample campaign scenario every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CampaignDirector` | `AutonomousAgent` | Decomposes the campaign brief, synthesises the merged plan, runs the brand-safety guardrail. | `CampaignWorkflow` | returns typed result to workflow |
| `WebsiteLauncher` | `AutonomousAgent` | Produces a website launch checklist and launch brief for the campaign. Seeded "channel tool" returns canned outputs. | `CampaignWorkflow` | — |
| `StrategyAdvisor` | `AutonomousAgent` | Produces a positioning strategy and messaging framework. | `CampaignWorkflow` | — |
| `CampaignWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, the guardrail. | `CampaignEndpoint`, `CampaignRequestConsumer` | `CampaignPlanEntity` |
| `CampaignPlanEntity` | `EventSourcedEntity` | Holds the plan's lifecycle (planning → in-progress → synthesised / degraded / blocked). | `CampaignWorkflow` | `CampaignView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted campaign brief for replay/audit. | `CampaignEndpoint`, `RequestSimulator` | `CampaignRequestConsumer` |
| `CampaignView` | `View` | List-of-plans read model. | `CampaignPlanEntity` events | `CampaignEndpoint` |
| `CampaignRequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `CampaignWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample campaign scenario every 60 s. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesised plan every 5 minutes for brand-quality scoring; emits a `PlanEvalScored` event. | scheduler | `CampaignPlanEntity` |
| `CampaignEndpoint` | `HttpEndpoint` | `/api/campaigns/*` — submit, get, list, SSE. | — | `CampaignView`, `RequestQueue`, `CampaignPlanEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record CampaignBriefRequest(String campaignName, String objective, String requestedBy) {}

record LaunchBrief(List<LaunchItem> launchItems, String channelRecommendation, Instant preparedAt) {}
record LaunchItem(String task, String priority, String rationale) {}

record StrategyFramework(String positioningStatement, List<String> messagingPillars, List<String> targetAudiences, Instant preparedAt) {}

record WorkScope(String launchScope, String strategyBrief) {}

record SynthesisedPlan(String executiveSummary, LaunchBrief launchBrief,
                       StrategyFramework strategyFramework,
                       String guardrailVerdict, Instant synthesisedAt) {}

record CampaignPlan(
    String planId,
    String campaignName,
    String objective,
    PlanStatus status,
    Optional<LaunchBrief> launchBrief,
    Optional<StrategyFramework> strategyFramework,
    Optional<SynthesisedPlan> synthesised,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PlanStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

### Events (on `CampaignPlanEntity`)

`PlanCreated`, `LaunchBriefAttached`, `StrategyFrameworkAttached`, `PlanSynthesised`, `PlanDegraded`, `PlanBlocked`, `PlanEvalScored`.

### Events (on `RequestQueue`)

`CampaignBriefSubmitted { planId, campaignName, objective, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/campaigns` — body `{ campaignName, objective }` → `{ planId }`. Starts a workflow.
- `GET /api/campaigns` — list all plans. Optional `?status=PLANNING|IN_PROGRESS|SYNTHESISED|DEGRADED|BLOCKED`.
- `GET /api/campaigns/{id}` — one plan.
- `GET /api/campaigns/sse` — server-sent events stream of every plan change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Marketing Agency Team"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a campaign brief, live list of plans with status pills, expand-row to see launch brief + strategy framework + synthesised executive summary + eval score.

Browser title: `<title>Akka Sample: Marketing Agency Team</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — brand-safety guardrail** (`before-agent-response` on `CampaignDirector`): vets the synthesised plan for off-brand copy, misleading claims, and non-compliant messaging before the plan is returned. Blocking. Failure → `BLOCKED`.

## 9. Agent prompts

- `CampaignDirector` → `prompts/campaign-director.md`. Decomposes the brief into work scopes; later synthesises results into the final marketing plan.
- `WebsiteLauncher` → `prompts/website-launcher.md`. Produces a launch checklist and brief; returns `LaunchBrief`.
- `StrategyAdvisor` → `prompts/strategy-advisor.md`. Produces a positioning strategy; returns `StrategyFramework`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a campaign brief; plan progresses PLANNING → IN_PROGRESS → SYNTHESISED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `WebsiteLauncher` timeout to 1 s); plan enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a guardrail failure (CampaignDirector returns off-brand content); plan enters BLOCKED.
4. **J4** — Wait after a successful synthesis; the plan's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named marketing-agency-team demonstrating the
delegation-supervisor-workers × sales-marketing cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-sales-marketing-marketing-agency-team.
Java package io.akka.samples.marketingagency. Akka 3.6.0. HTTP port 9404.

Components to wire (exactly):
- 3 AutonomousAgents:
  * CampaignDirector — definition() with capability(TaskAcceptance.of(SCOPE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/campaign-director.md. Returns WorkScope{launchScope, strategyBrief} for SCOPE
    and SynthesisedPlan{executiveSummary, launchBrief, strategyFramework,
    guardrailVerdict, synthesisedAt} for SYNTHESISE.
  * WebsiteLauncher — capability(TaskAcceptance.of(LAUNCH).maxIterationsPerTask(3)). System prompt
    from prompts/website-launcher.md. Returns LaunchBrief{launchItems: List<LaunchItem{task,
    priority, rationale}>, channelRecommendation, preparedAt}.
  * StrategyAdvisor — capability(TaskAcceptance.of(STRATEGISE).maxIterationsPerTask(2)). System
    prompt from prompts/strategy-advisor.md. Returns StrategyFramework{positioningStatement,
    messagingPillars: List<String>, targetAudiences: List<String>, preparedAt}.

- 1 Workflow CampaignWorkflow with steps:
  scopeStep -> [parallel] launchStep, strategiseStep -> joinStep -> synthesiseStep ->
  guardrailStep -> emitStep.
  scopeStep calls forAutonomousAgent(CampaignDirector.class, SCOPE).
  launchStep and strategiseStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(CampaignWorkflow::launchStep, ofSeconds(60)) and
  stepTimeout(CampaignWorkflow::strategiseStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with PlanDegraded.
  synthesiseStep calls forAutonomousAgent(CampaignDirector.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep runs the deterministic vetter +
  LLM judge on the synthesised content; on failure, end with PlanBlocked. WorkflowSettings is
  nested inside Workflow — no import.

- 1 EventSourcedEntity CampaignPlanEntity holding state CampaignPlan{planId, campaignName,
  objective, PlanStatus, Optional<LaunchBrief> launchBrief, Optional<StrategyFramework>
  strategyFramework, Optional<SynthesisedPlan> synthesised, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. PlanStatus enum: PLANNING, IN_PROGRESS, SYNTHESISED,
  DEGRADED, BLOCKED. Events: PlanCreated, LaunchBriefAttached, StrategyFrameworkAttached,
  PlanSynthesised, PlanDegraded, PlanBlocked, PlanEvalScored. Commands: createPlan,
  attachLaunchBrief, attachStrategyFramework, synthesise, degrade, block, recordEval, getPlan.
  emptyState() returns CampaignPlan.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command submitBrief(campaignName, objective, requestedBy)
  emitting CampaignBriefSubmitted{planId, campaignName, objective, requestedBy, submittedAt}.

- 1 View CampaignView with row type CampaignPlanRow (mirrors CampaignPlan minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  CampaignPlanEntity events. ONE query getAllPlans SELECT * AS plans FROM campaign_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer CampaignRequestConsumer subscribed to RequestQueue events; on CampaignBriefSubmitted
  starts a CampaignWorkflow with the planId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/campaign-scenarios.jsonl and calls RequestQueue.submitBrief.
  * EvalSampler — every 5 minutes, queries CampaignView.getAllPlans, picks the oldest
    SYNTHESISED plan without an evalScore, runs a 1-5 brand-quality rubric judge over the
    synthesised content, then calls CampaignPlanEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * CampaignEndpoint at /api with POST /campaigns, GET /campaigns, GET /campaigns/{id},
    GET /campaigns/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- MarketingTasks.java declaring four Task<R> constants: SCOPE (WorkScope), LAUNCH (LaunchBrief),
  STRATEGISE (StrategyFramework), SYNTHESISE (SynthesisedPlan).
- Domain records WorkScope, LaunchItem, LaunchBrief, StrategyFramework, SynthesisedPlan.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9404 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/campaign-scenarios.jsonl with 8 canned scenario lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control (G1 brand-safety guardrail
  before-agent-response) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  marketing-plan-generation, decisions.authority_level = recommend-only,
  data.data_classes.pii = false, capabilities.* = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/campaign-director.md, prompts/website-launcher.md, prompts/strategy-advisor.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Marketing Agency Team", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Marketing Agency Team</title>. No
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
  src/main/resources/mock-responses/<agent-name>.json (campaign-director.json,
  website-launcher.json, strategy-advisor.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    campaign-director.json — list of either WorkScope or SynthesisedPlan objects.
      4–6 WorkScope entries (launchScope + strategyBrief pairs describing different
      channel/product launch scenarios) and 4–6 SynthesisedPlan entries (each with
      a 60–120 word executive summary, a realistic LaunchBrief, a StrategyFramework,
      guardrailVerdict = "ok").
    website-launcher.json — 4–6 LaunchBrief entries, each with 3–6 LaunchItems
      (task, priority HIGH/MEDIUM/LOW, rationale) and a channelRecommendation.
    strategy-advisor.json — 4–6 StrategyFramework entries, each with a one-sentence
      positioningStatement, 3–5 messagingPillars, and 2–4 targetAudiences.
- A MockModelProvider.seedFor(planId) helper makes the selection deterministic
  per plan id so the same plan produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion MarketingTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9404 in application.conf.
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
