# SPEC — campaign-optimizer-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Campaign Optimizer.
**One-line pitch:** Submit a campaign brief; a Planner analyzes goals, builds an execution plan on a campaign ledger, dispatches each step to specialist agents (copy, audience, publish, performance), requires marketer approval before going live, and re-optimizes when KPIs miss targets.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to a marketing campaign lifecycle. The Planner owns two ledgers — a **campaign ledger** (goals, target audience, channel plan, brand constraints, current dispatch) and a **run ledger** (each step's attempt count, verdict, metrics snapshot, blockers). Each loop iteration the Planner reads both ledgers, picks the next specialist, and either continues, replans, completes, or fails.

The blueprint also demonstrates three governance mechanisms wired into that loop:

- a **before-agent-response guardrail** that checks every piece of generated copy against a brand-voice policy before it can advance to the publish step,
- a **human-in-the-loop approval gate** that holds the campaign in `AWAITING_APPROVAL` until a marketer explicitly approves or rejects — no assets go live without sign-off,
- a **periodic performance monitor** that wakes after the campaign goes `LIVE`, reads simulated KPI fixtures, and drives a re-optimization loop when metrics fall below the declared thresholds.

## 3. User-facing flows

The user opens the App UI tab and submits a campaign brief via the form.

1. The system creates a `Campaign` record in `PLANNING` and starts a `CampaignWorkflow`.
2. The Planner drafts a `CampaignLedger { goals, targetAudience, channelPlan, brandConstraints, dispatch }` and emits `CampaignPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Planner reads both ledgers and proposes a `DispatchDecision { specialist, step, rationale }`.
   - The chosen specialist runs the step and returns a typed `StepResult`.
   - The **before-agent-response guardrail** intercepts copy results from `CopyWriterAgent`; on policy violation it records a `StepBlocked` entry and asks the Planner to revise.
   - The workflow appends a `RunEntry { specialist, step, attempt, verdict, output, metricsSnapshot }` to the run ledger.
4. The Planner decides on each tick: `CONTINUE`, `REPLAN`, `SUBMIT_FOR_APPROVAL`, or `FAIL`.
5. On `SUBMIT_FOR_APPROVAL`, the workflow moves to `AWAITING_APPROVAL` and blocks on `ApprovalEntity`. The marketer's **human-in-the-loop** decision either advances the workflow to `LIVE` or ends it in `REJECTED`.
6. Once `LIVE`, the **performance monitor** (TimedAction) ticks every 60 s, reads KPI fixtures, and if KPIs miss threshold emits `PerformanceAlertFired`. The workflow re-enters the executor loop with a revised plan.
7. On natural completion (KPIs met), the Planner produces a `CampaignReport` and emits `CampaignCompleted`. Status moves to `COMPLETED`.

A `CampaignSimulator` (TimedAction) drips a sample campaign brief every 120 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains campaign ledger; reads run ledger. Produces `CampaignReport` on completion. | `CampaignWorkflow` | returns typed result to workflow |
| `CopyWriterAgent` | `AutonomousAgent` | Drafts email subject lines, ad headlines, and body copy from seeded brand-voice fixtures. | `CampaignWorkflow` | — |
| `AudienceSegmenterAgent` | `AutonomousAgent` | Selects audience segments from seeded CRM cohort fixtures. Returns `AudienceSelection`. | `CampaignWorkflow` | — |
| `AssetPublisherAgent` | `AutonomousAgent` | Simulates publishing assets to a channel endpoint from seeded channel fixtures. | `CampaignWorkflow` | — |
| `PerformanceAnalystAgent` | `AutonomousAgent` | Reads simulated KPI fixtures and returns a `PerformanceAssessment`. | `CampaignWorkflow` | — |
| `CampaignWorkflow` | `Workflow` | Drives the plan → approval-gate → execute → record → evaluate loop, plus replan and halt branches. | `CampaignEndpoint`, `CampaignRequestConsumer` | `CampaignEntity`, `ApprovalEntity` |
| `CampaignEntity` | `EventSourcedEntity` | Holds the campaign's lifecycle, campaign ledger, run ledger, and final report. | `CampaignWorkflow` | `CampaignView` |
| `ApprovalEntity` | `EventSourcedEntity` | Holds the marketer's approval decision. Keyed by `campaignId`. | `CampaignEndpoint` (marketer action), `CampaignWorkflow` (polls) | `CampaignWorkflow` |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted campaign briefs. | `CampaignEndpoint`, `CampaignSimulator` | `CampaignRequestConsumer` |
| `CampaignView` | `View` | List-of-campaigns read model for the UI. | `CampaignEntity` events | `CampaignEndpoint` |
| `CampaignRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `CampaignWorkflow` per submission. | `RequestQueue` events | `CampaignWorkflow` |
| `CampaignSimulator` | `TimedAction` | Every 120 s, reads a line from `sample-events/campaign-briefs.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `PerformanceMonitor` | `TimedAction` | Every 60 s, reads KPI fixtures for campaigns in `LIVE` status and emits `PerformanceAlertFired` when KPIs miss threshold. | scheduler | `CampaignEntity` |
| `CampaignEndpoint` | `HttpEndpoint` | `/api/campaigns/*` — submit, get, list, SSE, approve, reject. | — | `CampaignView`, `RequestQueue`, `CampaignEntity`, `ApprovalEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record CampaignBrief(String goal, String targetAudience, String channels, String requestedBy) {}

record CampaignLedger(
    List<String> goals,
    List<String> targetAudience,
    List<String> channelPlan,
    List<String> brandConstraints,
    Optional<DispatchDecision> currentDispatch
) {}

record DispatchDecision(
    SpecialistKind specialist,
    String step,
    String rationale
) {}

record StepResult(
    SpecialistKind specialist,
    String step,
    boolean ok,
    String output,
    Optional<String> errorReason
) {}

record MetricsSnapshot(
    Optional<Double> openRate,
    Optional<Double> clickRate,
    Optional<Double> conversionRate,
    Optional<Long> impressions
) {}

record RunEntry(
    int attempt,
    SpecialistKind specialist,
    String step,
    RunVerdict verdict,
    String output,
    MetricsSnapshot metricsSnapshot,
    Optional<String> blocker,
    Instant recordedAt
) {}

record RunLedger(List<RunEntry> entries) {}

record AudienceSelection(String segmentName, long estimatedReach, List<String> cohortTags) {}

record PerformanceAssessment(
    boolean kpisMet,
    MetricsSnapshot current,
    MetricsSnapshot threshold,
    List<String> recommendations
) {}

record CampaignReport(
    String summary,
    List<String> highlights,
    MetricsSnapshot finalMetrics,
    Instant producedAt
) {}

record Campaign(
    String campaignId,
    String goal,
    CampaignStatus status,
    Optional<CampaignLedger> ledger,
    Optional<RunLedger> runLedger,
    Optional<CampaignReport> report,
    Optional<String> failureReason,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

record ApprovalDecision(String campaignId, boolean approved, String decidedBy, String note, Instant decidedAt) {}

enum SpecialistKind { COPY, AUDIENCE, PUBLISH, PERFORMANCE }
enum RunVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, KPI_MISS }
enum CampaignStatus { PLANNING, EXECUTING, AWAITING_APPROVAL, LIVE, COMPLETED, FAILED, REJECTED }
```

### Events (`CampaignEntity`)

`CampaignCreated`, `CampaignPlanned`, `StepDispatched`, `StepBlocked`, `StepRecorded`, `LedgerRevised`, `ApprovalRequested`, `CampaignApproved`, `CampaignRejected`, `CampaignWentLive`, `PerformanceAlertFired`, `CampaignCompleted`, `CampaignFailed`.

### Events (`ApprovalEntity`)

`ApprovalGranted { campaignId, decidedBy, note, grantedAt }`, `ApprovalDenied { campaignId, decidedBy, note, deniedAt }`.

### Events (`RequestQueue`)

`CampaignSubmitted { campaignId, goal, targetAudience, channels, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/campaigns` — body `{ goal, targetAudience, channels, requestedBy? }` → `202 { campaignId }`. Starts a workflow.
- `GET /api/campaigns` — list all campaigns. Optional `?status=...`.
- `GET /api/campaigns/{id}` — one campaign (full ledgers + report).
- `GET /api/campaigns/sse` — server-sent events stream of every campaign change.
- `POST /api/campaigns/{id}/approve` — body `{ decidedBy, note }` → `200`. Records marketer approval.
- `POST /api/campaigns/{id}/reject` — body `{ decidedBy, note }` → `200`. Records marketer rejection.
- `GET /api/campaigns/{id}/approval` — `{ approved?, decidedBy?, note?, decidedAt? }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Campaign Optimizer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a campaign brief, approval controls for `AWAITING_APPROVAL` campaigns, live list of campaigns with status pills, expand-row to see the campaign ledger, the run ledger entries, and the final report.

Browser title: `<title>Akka Sample: Campaign Optimizer</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`before-agent-response` on `CopyWriterAgent`): every piece of generated copy is checked against a brand-voice policy before it can advance in the workflow. The policy rejects copy that contains competitor brand names, superlative regulatory claims ("best", "guaranteed"), or prohibited content categories. Blocking. Failure → `StepBlocked` entry + replan request to `PlannerAgent`.
- **H1 — human-in-the-loop approval** (`hitl`, flavor `application`): after the Planner emits `SUBMIT_FOR_APPROVAL`, the workflow blocks on `ApprovalEntity` and surfaces an approval card in the UI. A marketer must explicitly approve (campaign moves to `LIVE`) or reject (campaign moves to `REJECTED`). No assets are published until approval is granted.
- **E1 — periodic performance monitor** (`eval-periodic`, flavor `performance-monitor`): once the campaign is `LIVE`, `PerformanceMonitor` ticks every 60 s, reads KPI fixtures for all live campaigns, and emits `PerformanceAlertFired` when open rate, click rate, or conversion rate falls below the declared threshold. The workflow re-enters the executor loop for re-optimization; `PerformanceAnalystAgent` proposes recommendations.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Maintains both ledgers; decides next step; produces `CampaignReport`.
- `CopyWriterAgent` → `prompts/copy-writer.md`. Drafts copy from brand-voice fixtures.
- `AudienceSegmenterAgent` → `prompts/audience-segmenter.md`. Selects segments from CRM cohort fixtures.
- `AssetPublisherAgent` → `prompts/asset-publisher.md`. Simulates channel publishing.
- `PerformanceAnalystAgent` → `prompts/performance-analyst.md`. Reads KPI fixtures; returns assessment.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Run a product-launch email campaign targeting enterprise buyers in North America." Campaign progresses `PLANNING → EXECUTING → AWAITING_APPROVAL → LIVE → COMPLETED` within ~4 minutes with marketer approval. UI reflects each transition via SSE.
2. **J2** — Submit a brief whose copy step produces a competitor brand mention. The guardrail blocks the step; the planner replans; the campaign either completes via revised copy or fails after exhausting the replan budget.
3. **J3** — Submit a brief and click **Reject** on the approval card. The campaign moves to `REJECTED` immediately; no `AssetPublisher` steps have run.
4. **J4** — Submit a brief; once the campaign is `LIVE`, a KPI fixture returns open rate 0.03 (below threshold 0.08). `PerformanceMonitor` fires `PerformanceAlertFired`; the workflow replans with revised copy or audience.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named campaign-optimizer-loop demonstrating the
planner-executor × sales-marketing cell. Requires a marketing platform
credential for the integration tier.
Maven group io.akka.samples. Maven artifact
planner-executor-sales-marketing-campaign-optimizer-loop. Java package
io.akka.samples.campaignoptimizer. Akka 3.6.0. HTTP port 9580.

Components to wire (exactly):
- 5 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_CAMPAIGN).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN_CAMPAIGN returns CampaignLedger.
    DECIDE returns a NextStep tagged union (Continue(DispatchDecision) |
    Replan(CampaignLedger) | SubmitForApproval | Fail(String)).
    COMPOSE_REPORT returns CampaignReport.
  * CopyWriterAgent — capability(TaskAcceptance.of(WRITE_COPY).maxIterationsPerTask(2)).
    Prompt from prompts/copy-writer.md. Returns StepResult with output
    holding generated copy text.
  * AudienceSegmenterAgent — capability(TaskAcceptance.of(SELECT_AUDIENCE).maxIterationsPerTask(2)).
    Prompt from prompts/audience-segmenter.md. Returns StepResult with
    output holding AudienceSelection serialized as JSON.
  * AssetPublisherAgent — capability(TaskAcceptance.of(PUBLISH_ASSETS).maxIterationsPerTask(2)).
    Prompt from prompts/asset-publisher.md. Returns StepResult.
  * PerformanceAnalystAgent — capability(TaskAcceptance.of(ANALYZE_PERFORMANCE).maxIterationsPerTask(2)).
    Prompt from prompts/performance-analyst.md. Returns StepResult with
    output holding PerformanceAssessment serialized as JSON.

- 1 Workflow CampaignWorkflow with steps:
  planStep -> [loop entry] proposeStep -> guardrailStep ->
  dispatchStep -> recordStep -> decideStep
  -> [back to proposeStep, or to approvalGateStep, or to composeReportStep / failStep / rejectedStep].
  approvalGateStep polls ApprovalEntity.get until approved or denied;
  on approved transitions to goLiveStep (emits CampaignWentLive, CampaignApproved);
  on denied transitions to rejectedStep (emits CampaignRejected).
  goLiveStep emits CampaignWentLive then re-enters the performance loop.
  guardrailStep intercepts StepResult from CopyWriterAgent only; on policy
  violation records a StepBlocked entry via CampaignEntity.recordBlock and
  loops back to proposeStep.
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120), decideStep ofSeconds(45), composeReportStep ofSeconds(60),
    approvalGateStep ofSeconds(1800) (30 minutes for human review).
  defaultStepRecovery(maxRetries(2).failoverTo(CampaignWorkflow::error)).

- 1 EventSourcedEntity CampaignEntity holding Campaign state. emptyState()
  returns Campaign.initial("", null). Commands: createCampaign, recordPlan,
  recordDispatch, recordBlock, recordStep, reviseLedger, requestApproval,
  approveCampaign, rejectCampaign, goLive, firePerformanceAlert,
  completeCampaign, failCampaign, getCampaign. Events as listed in SPEC §5.

- 1 EventSourcedEntity ApprovalEntity keyed by campaignId. State
  Approval{String campaignId, Optional<ApprovalDecision> decision}. Commands:
  grant(decidedBy, note), deny(decidedBy, note), get. Events:
  ApprovalGranted, ApprovalDenied.

- 1 EventSourcedEntity RequestQueue with command enqueueCampaign(campaignId,
  goal, targetAudience, channels, requestedBy) emitting CampaignSubmitted.

- 1 View CampaignView with row type CampaignRow (mirror of Campaign minus
  heavy ledger payloads — truncate to last 3 run entries plus counts; the
  UI fetches the full campaign by id on click). Table updater consumes
  CampaignEntity events. ONE query getAllCampaigns SELECT * AS campaigns
  FROM campaign_view. No WHERE status filter — caller filters client-side
  (Lesson 2).

- 1 Consumer CampaignRequestConsumer subscribed to RequestQueue events; on
  CampaignSubmitted starts a CampaignWorkflow with campaignId as the
  workflow id.

- 2 TimedActions:
  * CampaignSimulator — every 120s, reads next line from
    src/main/resources/sample-events/campaign-briefs.jsonl and calls
    RequestQueue.enqueueCampaign.
  * PerformanceMonitor — every 60s, queries CampaignView.getAllCampaigns,
    filters LIVE campaigns, reads KPI fixtures from
    src/main/resources/sample-data/kpi-fixtures.jsonl, emits
    PerformanceAlertFired via CampaignEntity.firePerformanceAlert when
    open rate < 0.08 or click rate < 0.02 or conversion rate < 0.005.

- 2 HttpEndpoints:
  * CampaignEndpoint at /api with POST /campaigns, GET /campaigns,
    GET /campaigns/{id}, GET /campaigns/sse,
    POST /campaigns/{id}/approve, POST /campaigns/{id}/reject,
    GET /campaigns/{id}/approval, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN_CAMPAIGN
  (resultConformsTo CampaignLedger), DECIDE (NextStep), COMPOSE_REPORT
  (CampaignReport).
- SpecialistTasks.java declaring four Task<R> constants: WRITE_COPY,
  SELECT_AUDIENCE, PUBLISH_ASSETS, ANALYZE_PERFORMANCE (all
  resultConformsTo StepResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface
  with permits Continue, Replan, SubmitForApproval, Fail.
- application/CopyGuardrail.java — deterministic brand-voice checker.
  Reject if output contains any known competitor brand name (from a
  configurable allow-list in application.conf), contains superlatives
  paired with regulatory framing ("guaranteed results", "proven to cure",
  "legally required to"), or contains content from a prohibited-topics
  list (hate speech markers, political party names). Replacements:
  blocker = "COPY_VIOLATION:<reason>".
- application/KpiThresholds.java — threshold constants: OPEN_RATE_MIN =
  0.08, CLICK_RATE_MIN = 0.02, CONVERSION_RATE_MIN = 0.005. Used by
  PerformanceMonitor.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9580 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  marketing.platform.api-key = ${?MARKETING_PLATFORM_API_KEY}.
- src/main/resources/sample-events/campaign-briefs.jsonl with 8 canned
  campaign briefs spanning email, social, display, and content channels.
- src/main/resources/sample-data/brand-voice-fixtures.jsonl — 12 canned
  copy examples (approved and rejected pairs) used by CopyWriterAgent.
- src/main/resources/sample-data/crm-cohort-fixtures.jsonl — 8 audience
  segments with estimated reach and cohort tags used by
  AudienceSegmenterAgent.
- src/main/resources/sample-data/channel-fixtures.jsonl — 6 channel
  endpoint definitions used by AssetPublisherAgent.
- src/main/resources/sample-data/kpi-fixtures.jsonl — 10 KPI snapshots,
  including two with open rate < 0.08 to trigger the performance alert
  in J4.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies for metadata endpoint).
- eval-matrix.yaml at the project root with 3 controls (G1, H1, E1) and
  a matching simplified_view list.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance.
- prompts/planner.md, prompts/copy-writer.md, prompts/audience-segmenter.md,
  prompts/asset-publisher.md, prompts/performance-analyst.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Campaign Optimizer",
  one-line pitch, prerequisites (including the integration form's
  host-software requirement: Marketing platform credential), generate-
  the-system, what-you-get, customise-before-generating, what-gets-
  validated, license. NO Configuration section. NO governance-mechanisms
  section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated
  from risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix
  (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows), App UI (form + approval controls for
  AWAITING_APPROVAL campaigns + live list with status pills and
  expand-on-click for ledgers and report). Browser title exactly:
  <title>Akka Sample: Campaign Optimizer</title>. No subtitle on
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java with
  per-agent dispatch. Mock-response files:
    planner.json — three lists keyed by task id:
      "PLAN_CAMPAIGN" → 4–6 CampaignLedger entries.
      "DECIDE" → 4–6 NextStep entries covering Continue, Replan,
        SubmitForApproval, Fail.
      "COMPOSE_REPORT" → 4–6 CampaignReport entries with 60–120 word
        summaries.
    copy-writer.json — 6 StepResult entries, ok=true, output fields are
      generated copy samples. ONE entry must include a competitor brand
      name (e.g., "AcmeCorp") to exercise J2.
    audience-segmenter.json — 6 StepResult entries with AudienceSelection
      JSON in output.
    asset-publisher.json — 5 StepResult entries simulating channel publish
      confirmations.
    performance-analyst.json — 5 StepResult entries; TWO entries have
      kpisMet=false (open rate < 0.08) to exercise J4.
- MockModelProvider.seedFor(campaignId) makes selection deterministic.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on planStep,
  proposeStep, dispatchStep, decideStep, composeReportStep,
  approvalGateStep.
- Lesson 6: Optional<T> for every nullable field on CampaignView row
  record and on the Campaign entity state.
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  SpecialistTasks.java.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9580 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Marketing platform credential" — never
  T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow above.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute.
  No zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
