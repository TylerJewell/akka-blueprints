# SPEC — marketing-strategy-team-marketplace

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Marketing Strategy Team.
**One-line pitch:** Submit a campaign objective; a director delegates market research, audience targeting, messaging, and channel planning to four specialist agents in parallel, then assembles a compliant campaign brief.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to four AutonomousAgents in parallel, gathers their results, and asks a fifth AutonomousAgent to assemble a unified campaign brief. The blueprint also demonstrates a **before-agent-response guardrail** that vets all marketing claims for brand and legal compliance before the brief is returned, and **eval-event** governance that samples the director's assembly decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a campaign objective via the form.

1. The system creates a `CampaignBrief` record in `PLANNING` and starts a `CampaignWorkflow`.
2. The CampaignDirector decomposes the objective into four parallel work items: a market-research query, an audience-targeting question, a messaging brief, and a channel-planning directive.
3. The workflow forks: all four agents run concurrently. Each returns a typed payload.
4. The CampaignDirector assembles the four payloads into an `AssembledBrief { executiveSummary, marketInsights, audienceSegments, keyMessages, channelPlan, complianceVerdict }`.
5. A before-agent-response guardrail vets the assembled brief for brand and legal compliance; if it fails, the brief moves to `BLOCKED`. Otherwise, the brief moves to `ASSEMBLED`.
6. If any worker times out after 60 seconds, the workflow short-circuits: it asks the CampaignDirector to assemble from whichever sides returned, and the brief enters `DEGRADED`.

A `CampaignSimulator` (TimedAction) drips a sample campaign objective every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CampaignDirector` | `AutonomousAgent` | Decomposes the objective, assembles the merged brief, runs the compliance guardrail. | `CampaignWorkflow` | returns typed result to workflow |
| `MarketResearcher` | `AutonomousAgent` | Gathers market insights and competitive context. Seeded tools return canned data. | `CampaignWorkflow` | — |
| `AudienceTargeter` | `AutonomousAgent` | Defines audience segments and persona profiles. | `CampaignWorkflow` | — |
| `MessageStrategist` | `AutonomousAgent` | Drafts key messages and brand-aligned copy direction. | `CampaignWorkflow` | — |
| `ChannelPlanner` | `AutonomousAgent` | Selects and prioritises marketing channels with rationale. | `CampaignWorkflow` | — |
| `CampaignWorkflow` | `Workflow` | Coordinates the parallel fan-out, the assembly, the guardrail. | `CampaignEndpoint`, `CampaignRequestConsumer` | `CampaignBriefEntity` |
| `CampaignBriefEntity` | `EventSourcedEntity` | Holds the brief's lifecycle (planning → in-progress → assembled / degraded / blocked). | `CampaignWorkflow` | `CampaignView` |
| `ObjectiveQueue` | `EventSourcedEntity` | Logs each submitted objective for replay/audit. | `CampaignEndpoint`, `CampaignSimulator` | `CampaignRequestConsumer` |
| `CampaignView` | `View` | List-of-briefs read model. | `CampaignBriefEntity` events | `CampaignEndpoint` |
| `CampaignRequestConsumer` | `Consumer` | Listens to `ObjectiveQueue` events and starts a workflow per submission. | `ObjectiveQueue` events | `CampaignWorkflow` |
| `CampaignSimulator` | `TimedAction` | Drips a sample objective every 60 s. | scheduler | `ObjectiveQueue` |
| `EvalSampler` | `TimedAction` | Samples one assembled brief every 5 minutes for eval scoring; emits a `CampaignEvalScored` event. | scheduler | `CampaignBriefEntity` |
| `CampaignEndpoint` | `HttpEndpoint` | `/api/campaigns/*` — submit, get, list, SSE. | — | `CampaignView`, `ObjectiveQueue`, `CampaignBriefEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ObjectiveRequest(String objective, String requestedBy) {}

record MarketInsightsBundle(List<MarketInsight> insights, Instant gatheredAt) {}
record MarketInsight(String headline, String source, String detail) {}

record AudienceProfile(List<Segment> segments, String primaryPersona, Instant profiledAt) {}
record Segment(String name, String description, List<String> characteristics) {}

record MessagingGuide(String coreMessage, List<String> supportingMessages,
                     String toneGuidance, Instant createdAt) {}

record ChannelPlan(List<ChannelRecommendation> channels, String rationale, Instant createdAt) {}
record ChannelRecommendation(String channel, String priority, String justification) {}

record StrategyPlan(String marketQuery, String audienceQuestion,
                   String messagingBrief, String channelDirective) {}

record AssembledBrief(String executiveSummary, MarketInsightsBundle marketInsights,
                     AudienceProfile audienceProfile, MessagingGuide messagingGuide,
                     ChannelPlan channelPlan, String complianceVerdict, Instant assembledAt) {}

record CampaignBrief(
    String briefId,
    String objective,
    BriefStatus status,
    Optional<MarketInsightsBundle> marketInsights,
    Optional<AudienceProfile> audienceProfile,
    Optional<MessagingGuide> messagingGuide,
    Optional<ChannelPlan> channelPlan,
    Optional<AssembledBrief> assembled,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BriefStatus { PLANNING, IN_PROGRESS, ASSEMBLED, DEGRADED, BLOCKED }
```

### Events (on `CampaignBriefEntity`)

`BriefCreated`, `MarketInsightsAttached`, `AudienceProfileAttached`, `MessagingGuideAttached`,
`ChannelPlanAttached`, `BriefAssembled`, `BriefDegraded`, `BriefBlocked`, `CampaignEvalScored`.

### Events (on `ObjectiveQueue`)

`ObjectiveSubmitted { briefId, objective, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/campaigns` — body `{ objective }` → `{ briefId }`. Starts a workflow.
- `GET /api/campaigns` — list all briefs. Optional `?status=PLANNING|IN_PROGRESS|ASSEMBLED|DEGRADED|BLOCKED`.
- `GET /api/campaigns/{id}` — one brief.
- `GET /api/campaigns/sse` — server-sent events stream of every brief change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Marketing Strategy Team"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an objective, live list of briefs with status pills, expand-row to see market insights, audience profile, messaging guide, channel plan, assembled summary, and eval score.

Browser title: `<title>Akka Sample: Marketing Strategy Team</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — compliance guardrail** (`before-agent-response` on `CampaignDirector`): vets the assembled brief for brand violations and legally impermissible marketing claims. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one assembled brief every 5 minutes and emits a `CampaignEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `CampaignDirector` → `prompts/campaign-director.md`. Decomposes the objective into strategy work items; later assembles all worker outputs into the final campaign brief.
- `MarketResearcher` → `prompts/market-researcher.md`. Gathers market insights; returns `MarketInsightsBundle`.
- `AudienceTargeter` → `prompts/audience-targeter.md`. Defines audience segments; returns `AudienceProfile`.
- `MessageStrategist` → `prompts/message-strategist.md`. Crafts key messages; returns `MessagingGuide`.
- `ChannelPlanner` → `prompts/channel-planner.md`. Recommends channels; returns `ChannelPlan`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an objective; brief progresses PLANNING → IN_PROGRESS → ASSEMBLED within 120 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `MarketResearcher` timeout to 1 s); brief enters DEGRADED with whichever partial outputs came back.
3. **J3** — Inject a guardrail failure (Director returns a claim with a prohibited term); brief enters BLOCKED.
4. **J4** — Wait after a successful assembly; the brief's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named marketing-strategy-team-marketplace demonstrating the
delegation-supervisor-workers × sales-marketing cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-sales-marketing-marketing-strategy-team-marketplace.
Java package io.akka.samples.marketingstrategyteam. Akka 3.6.0. HTTP port 9476.

Components to wire (exactly):
- 5 AutonomousAgents:
  * CampaignDirector — definition() with capability(TaskAcceptance.of(PLAN_CAMPAIGN).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(ASSEMBLE_BRIEF).maxIterationsPerTask(3)). System prompt loaded
    from prompts/campaign-director.md. Returns StrategyPlan{marketQuery, audienceQuestion, messagingBrief,
    channelDirective} for PLAN_CAMPAIGN and AssembledBrief{executiveSummary, marketInsights,
    audienceProfile, messagingGuide, channelPlan, complianceVerdict, assembledAt} for ASSEMBLE_BRIEF.
  * MarketResearcher — capability(TaskAcceptance.of(RESEARCH_MARKET).maxIterationsPerTask(3)). System
    prompt from prompts/market-researcher.md. Returns MarketInsightsBundle{insights: List<MarketInsight
    {headline, source, detail}>, gatheredAt}.
  * AudienceTargeter — capability(TaskAcceptance.of(TARGET_AUDIENCE).maxIterationsPerTask(2)). System
    prompt from prompts/audience-targeter.md. Returns AudienceProfile{segments: List<Segment{name,
    description, characteristics}>, primaryPersona, profiledAt}.
  * MessageStrategist — capability(TaskAcceptance.of(CRAFT_MESSAGING).maxIterationsPerTask(2)). System
    prompt from prompts/message-strategist.md. Returns MessagingGuide{coreMessage, supportingMessages:
    List<String>, toneGuidance, createdAt}.
  * ChannelPlanner — capability(TaskAcceptance.of(PLAN_CHANNELS).maxIterationsPerTask(2)). System prompt
    from prompts/channel-planner.md. Returns ChannelPlan{channels: List<ChannelRecommendation{channel,
    priority, justification}>, rationale, createdAt}.

- 1 Workflow CampaignWorkflow with steps:
  planStep -> [parallel] researchStep, targetStep, messagingStep, channelStep -> joinStep
  -> assembleStep -> guardrailStep -> emitStep.
  planStep calls forAutonomousAgent(CampaignDirector.class, PLAN_CAMPAIGN).
  researchStep, targetStep, messagingStep, and channelStep run in parallel (CompletionStage zip
  chaining all four); each governed by WorkflowSettings.builder()
  .stepTimeout(CampaignWorkflow::researchStep, ofSeconds(60))
  .stepTimeout(CampaignWorkflow::targetStep, ofSeconds(60))
  .stepTimeout(CampaignWorkflow::messagingStep, ofSeconds(60))
  .stepTimeout(CampaignWorkflow::channelStep, ofSeconds(60)). On any timeout, transition to a
  degradeStep that calls assembleStep with whichever sides returned, then ends with BriefDegraded.
  assembleStep calls forAutonomousAgent(CampaignDirector.class, ASSEMBLE_BRIEF) with all four
  inputs; give assembleStep a 120s stepTimeout. guardrailStep runs a deterministic brand-term
  vetter plus LLM judge on the assembled content; on failure, end with BriefBlocked.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity CampaignBriefEntity holding state CampaignBrief{briefId, objective,
  BriefStatus, Optional<MarketInsightsBundle> marketInsights, Optional<AudienceProfile> audienceProfile,
  Optional<MessagingGuide> messagingGuide, Optional<ChannelPlan> channelPlan,
  Optional<AssembledBrief> assembled, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. BriefStatus enum: PLANNING, IN_PROGRESS, ASSEMBLED,
  DEGRADED, BLOCKED. Events: BriefCreated, MarketInsightsAttached, AudienceProfileAttached,
  MessagingGuideAttached, ChannelPlanAttached, BriefAssembled, BriefDegraded, BriefBlocked,
  CampaignEvalScored. Commands: createBrief, attachMarketInsights, attachAudienceProfile,
  attachMessagingGuide, attachChannelPlan, assemble, degrade, block, recordEval, getBrief.
  emptyState() returns CampaignBrief.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity ObjectiveQueue with command enqueueObjective(objective, requestedBy)
  emitting ObjectiveSubmitted{briefId, objective, requestedBy, submittedAt}.

- 1 View CampaignView with row type CampaignBriefRow (mirrors CampaignBrief minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  CampaignBriefEntity events. ONE query getAllBriefs SELECT * AS briefs FROM campaign_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer CampaignRequestConsumer subscribed to ObjectiveQueue events; on ObjectiveSubmitted
  starts a CampaignWorkflow with the briefId as the workflow id.

- 2 TimedActions:
  * CampaignSimulator — every 60s, reads next line from
    src/main/resources/sample-events/campaign-objectives.jsonl and calls
    ObjectiveQueue.enqueueObjective.
  * EvalSampler — every 5 minutes, queries CampaignView.getAllBriefs, picks the oldest
    ASSEMBLED brief without an evalScore, runs a 1–5 rubric judge over the assembled
    content, then calls CampaignBriefEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * CampaignEndpoint at /api with POST /campaigns, GET /campaigns, GET /campaigns/{id},
    GET /campaigns/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- CampaignTasks.java declaring five Task<R> constants: PLAN_CAMPAIGN (StrategyPlan),
  RESEARCH_MARKET (MarketInsightsBundle), TARGET_AUDIENCE (AudienceProfile),
  CRAFT_MESSAGING (MessagingGuide), PLAN_CHANNELS (ChannelPlan), ASSEMBLE_BRIEF (AssembledBrief).
- Domain records StrategyPlan, MarketInsight, MarketInsightsBundle, Segment, AudienceProfile,
  MessagingGuide, ChannelRecommendation, ChannelPlan, AssembledBrief.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9476 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/campaign-objectives.jsonl with 8 canned objective lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 compliance guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = campaign-
  strategy-assembly, decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false except content_generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/campaign-director.md, prompts/market-researcher.md, prompts/audience-targeter.md,
  prompts/message-strategist.md, prompts/channel-planner.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Marketing Strategy Team", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Marketing Strategy Team</title>. No
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
  market-researcher.json, audience-targeter.json, message-strategist.json,
  channel-planner.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    campaign-director.json — list of either StrategyPlan or AssembledBrief objects.
      4–6 StrategyPlan entries (marketQuery + audienceQuestion + messagingBrief +
      channelDirective quadruples) and 4–6 AssembledBrief entries (each with an
      80–150 word executiveSummary, populated market/audience/messaging/channel
      payloads, complianceVerdict = "compliant").
    market-researcher.json — 4–6 MarketInsightsBundle entries, each with 3–6 insights
      whose source values are believable (e.g., "Gartner 2024", "eMarketer Q1 2024",
      "unsourced — knowledge").
    audience-targeter.json — 4–6 AudienceProfile entries, each with 2–4 segments
      and a one-sentence primaryPersona description.
    message-strategist.json — 4–6 MessagingGuide entries, each with a coreMessage,
      3–5 supportingMessages, and a toneGuidance sentence.
    channel-planner.json — 4–6 ChannelPlan entries, each with 3–5 channel
      recommendations with priority (HIGH/MEDIUM/LOW) and a one-sentence
      justification per channel.
- A MockModelProvider.seedFor(briefId) helper makes the selection
  deterministic per brief id so the same brief produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  120s assembly); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion CampaignTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9476 in application.conf.
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
- Parallel workflow steps use CompletionStage zip chaining all four workers, NOT sequential calls.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
